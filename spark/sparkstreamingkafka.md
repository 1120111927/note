# Spark Streaming + Kafka #

对Kafka 0.10的Spark Streaming集成提供简单的并行性、Kafka分区和Spark分区之间1：1的对应关系以及对offset和元数据的访问。

## 包依赖 ##

```maven
groupId = org.apache.spark
artifactId = spark-streaming-kafka-0-10_2.12
version = 2.4.3
```

不要手动添加`org.apache.kafka`依赖项，`spark-streaming-kafka-0-10`具有传递依赖性，不同版本可能会产生不易诊断的不兼容问题。

## 创建DStream ##

```scala
import org.apache.kafka.clients.consumer.ConsumerRecord
import apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe

val kafkaParams = Map[String, Object](
    "bootstrap.servers" -> "localhost:9092, anotherhost:9092",
    "key.deserializer" -> classOf[StringDeserializer],
    "value.deserializer" -> classOf[StringDeserializer],
    "group.id" -> "use_a_separate_group_id_for_each_stream",
    "auto.offset.reset" -> "latest",
    "enable.auto.commit" -> (false: java.lang.Boolean)
)

val topics = Array("topicA", "topicB")
val stream = kafkaUtils.createDirectStream[String, String](
    streamingContext,
    PreferConsistend,
    Subscribe[String, String](topics, kafkaParams)
)

stream.map(record => (record.key, record.value))
```

Stream中元素类型为`ConsumerRecord`。

如果Spark时间片比默认Kafka会话心跳超时（30s）长的话，适当增大`heartbeat.interval.ms`和`session.timeout.ms`。时间片大于5min时，需要修改broker的`group.max.session.timeout.ms`。

kafka 0.10.0及之前只有`session.timeout.ms`，kafka 0.10.1及之后引入了`max.poll.interval.ms`，通过后台心跳线程将心跳和`poll()`的调用解耦，允许比心跳间隔更长的处理时间，及存在两个线程：心跳线程和处理线程。`session.timeout.ms`用于设置心跳线程超时，`max.poll.interval.ms`用于设置处理线程超时。

## 位置策略 ##

kafka新消费者API会预拉取消息到缓存区中，出于性能原因，Spark集成应该在executor上缓存消费者（而不是每个批次都重建），并且最好将分区分配到对应消费者所在的主机位置。

缓存消费者的默认最大值为64，如果分区数大于（64*executor数目），通过`spark.streaming.kafka.consumer.cache.maxCapacity`适当增大。

可以设置`spark.streaming.kafka.consumer.cache.enabled`为`false`来禁止缓存kafka消费者。

缓存由主题分区和`group.id`索引，每次调用`createDirectStream`时都要使用不同的`group.id`。

位置策略有以下三种：

+ `LocationStrategies.PreferConsistent`：一般使用该项，这将在可用executor间均匀分配分区
+ `LocationStrategies.PreferBrokers`：如果executor和kafka broker共享相同集群，使用该项，这将将分区分配到其对应的kafka leader上
+ `LocationStrategies.PreferFixed`：如果分区之间的负载有很大偏差，使用该项，将允许显示指定分区和主机的对应关系。没有指定的分区将使用PreferConsistent策略

## 消费者策略 ##

新kafka消费者API有多种方式指定话题，其中一些需要复杂的设置。ConsumerStrategies提供了一种抽象，即使从检查点恢复，Spark也可以获取正确配置的消费者。

+ `ConsumerStrategies.Subscribe`用于订阅固定的主题集合
+ `ConsumerStrategies.SubscribePattern`用于使用正则指定主题
+ `ConsumerStrategies.Assign`用于指定固定的分区集合

所有三种策略都有重载的构造器用于指定特定分区的起始offset。

和0.8集成不同，使用`Subscribe`或`SubscribePattern`将响应运行期间增加分区。

如果上面这些不满足特定的消费者设置需求，可以继承`ConsumerStrategy`进行自定义。

## 创建RDD ##

可以为指定的offset范围创建RDD：

```scala
val offsetRanges = Array(
    // topic, partition, inclusive starting offset, exclusive ending offset
    offsetRange("test", 0, 0, 100),
    offsetRange("test", 1, 0, 100)
)

val rdd = KafkaUtils.createRDD[String, String](sparkContext, kafkaParams, offsetRanges, preferConsistent)
```

此时不能使用`PreferBroker`，因为不使用流处理的话driver端没有消费者自动查找broker元数据。如需要使用`PreferFixed`和自定义的元数据查找。

## 获取Offset ##

```scala
stream.foreachRDD { rdd =>
    val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
    rdd.foreachPartition { iter =>
        val o: OffsetRange = offsetRanges(TaskContext.get.partitionId)
        println(s"${o.topic} ${o.partiiton} ${o.fromOffset} ${o.untilOffset}")
    }
}
```

`HasOffsetRanges`类型转换仅在`createDirectStream`后立即调用才能成功，而不是在一串方法后。RDD分区和Kafka分区间一对一的关系在调用需要shuffle或repartition的方法（比如`reduceByKey()`或`window()`）后都不再保持。

## 存储Offset ##

kafka传输语义在失败时依赖于offset是如何以及何时存储的。Spark输出操作是`at-least-once`，如果需要`exactly-once`语义，必须在幂式输出操作后保存offset，或者在原子事务中将offset和输出一起存储。该集成提供了三种存储offset的方式：

+ 检查点（Checkpoints）
+ Kafka自身
+ 自定义位置

从上往下，可靠性递增，代码复杂度递增。

### 检查点 ###

如果设置了Spark checkpointing，offset将会保存在检查点中。这要求输出操作必须是幂等的，否则不能保证`exactly-once`语义；通过事务操作也不能满足。另外，如果应用代码有改变将不能从检查点恢复，对于计划中的更新，可以将新旧代码同时运行来减少影响（输出操作必须是幂等的）。但是对于预料外的代码改变，将丢失数据，除非可以通过其他方法标识正确的起始偏移。

### kafka自身 ###

kafka提供了一个offset提交APi来将偏移存储到一个特殊的kafka主题中。新消费者默认将周期性的自动提交offset。这一般和实际需要不符，因为消费者成功拉取的消息可能并未在Spark输出操作中使用，导致不确定的语义。所以一般将`enable.auto.commit`设置为false。可以通过`commitAsync`API在成功存储输出后提交offset。和检查点相比，无论应用代码如何改变，kafka存储都可用。但是kafka不是事务的，输出仍需要幂等性。

```scala
stream.foreachRDD { rdd =>
    val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

    // 输出操作完成后
    stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
}
```

和`HashOffsetRanges`相同，`CanCommitOffset`仅在`createDirectStream`后立即调用才能成功，而不是在transformation后。`commitAsync`是线程安全的，但是为了保证清晰的语义必须在输出操作后执行。

### 自定义位置 ###

对于支持事务的数据存储，将结果和offset在同个事务中保存可以保证二者同步，即使在遭遇失败。回退事务将消除消息重复或丢失对结果的影响，提供了`exactly-once`语义。这对于输出为聚合的场景也有效，聚合操作经常难以具有幂等性。

```scala
// 从数据库中保存的offset开始
val fromOffsets = selectOffsetsFromYourDatabase.map { resultSet =>
    new TopicPartition(resultSet.string("topic"), resultSet.int("partition")) -> resultSet.long("offset")
}.toMap

val stream = kafkaUtils.createDirectStream[String, String](
    streamingContext,
    PreferConsistent,
    Assign[String, String](fromOffsets.keys.toList, kafkaParams, romOffsets)
)

stream.foreachRDD { rdd =>
    val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

    val results = yourCalculation(rdd)

    // begin your transaction

    // update results

    // update offsets where the end of existing offsets matches the beginning of this batch of ofsets

    // assert that offsets were updated correctly

    // end your transaction
}
```
