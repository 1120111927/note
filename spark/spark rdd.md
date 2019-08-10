# Spark RDD #

RDD（Resilient Distributed Dataset，弹性分布式数据集）是Spark的基本抽象，表示已被分区、不可变并能被并行操作的数据集合。

RDD的类型包含任何java类型、scala类型、python类型或者自定义的类型。

`org.apache.rdd.RDD`：

+ `final def partitions: Array[Partition]：以数组形式返回RDD的分区列表，考虑checkpoint`
+ `abstract def compute(split: Partition, context: TaskContext): Iterator[T]`：抽象成员，计算指定分区，由具体子类实现
+ `final def dependencies: Seq[Denpendency[_]]`：返回RDD的依赖列表，考虑checkpoint
+ `val partitioner: Option[Partitioner]`：子类可以覆盖该属性来指定其分区方式
+ `final def preferredLocations(split: Partition): Seq[String]`：返回计算分区的最佳节点位置，考虑checkpoint

## 原理 ##

`org.apache.spark.rdd.RDD`类包含了所有RDD的基本操作，比如`map`、`filter`、`persist`。底层实现上，RDD有五个特性：

+ 分区列表：通过分区列表可以找到一个RDD中包含的所有分区及其所在地址
+ 计算每个分片的函数
+ RDD的依赖关系
+ 可选，键值对RDD的分区器
+ 可选，数据最佳计算位置，实现数据局部性（data locality）

Spark中的所有调度和执行都基于这些属性。

**分区**：在Spark中所有数据（输入、中间结果、输出）都被表示为分区，分区是数据的逻辑划分，是并行处理的基本单元，RDD就是分区的集合。底层存储方式决定RDD分区都是不可变的，每种RDD变换都会生成新的分区，另外，分区的不可变性有利于故障恢复。分区来源于HDFS，默认就是分布式的。分区具有位置感知，用于实现数据局部性。RDD的`mapPartitions()`接口可以同时获取分区所有数据，而不是一次一行，用于进行按分区操作。分区由指定的partitioner决定，默认使用HashPartitioner，也可以自定义partitioner。分区有利于快速查找。

**依赖关系**：RDD间的依赖关系分为窄依赖（narrow dependency）和宽依赖（wide dependency）。如果RDD的每个分区至多只被一个子RDD分区使用，则称之为窄依赖；若被多个子RDD分区依赖，则称之为宽依赖。不同的操作依据其特性，可能会产生不同的依赖，比如map产生窄依赖，join产生宽依赖。窄依赖在通过节点上以管道形式执行多条指令。宽依赖则需要所有父分区可用且通过类MapReduce操作在多个节点间进行shuffle操作。窄依赖的故障恢复效率高，对于一个存在宽依赖的RDD运算图，如果一个节点失败，可能导致某个RDD的一些依赖所有父RDD的分区丢失，进而需要完全重新计算。用户对RDD执行Action操作时，调度器根据这个RDD的逻辑执行图构建执行阶段的物理执行图。每个阶段（stage）包含尽可能多的产生窄依赖的可以管道化执行的变换操作，阶段的边界是宽依赖需要的shuffle操作，或者任何已经计算好的分区。然后，调度器加载任务去计算缺失的分区直到计算出目标RDD。

**惰性求值**：Spark中的所有变换操作都是惰性的，即不立即进行计算，仅仅记录对数据集应用的变换，仅当动作需要向driver program返回结果时才开始执行计算。每个RDD都可以获取其依赖RDD的列表，第一个RDD依赖的RDD为Nil，计算RDD前会先计算其依赖的RDD，这种运行链使得RDD支持惰性求值。Spark的每种操作都创建RDD特定子类的实例，map操作产生MappedRDD，flatMap操作产生FlatMappedRDD，这使得RDD可以记录变换中使用操作的类型。

+ HadoopRDD：使用旧MapReduce接口（`org.apache.hadoop.mapred`）读取Hadoop中数据
+ newHadoopRDD：使用新MapReduce接口（`org.apache.hadoop.mapreduce`）读取Hadoop中数据
+ JdbcRDD：通过JDBC连接执行SQL查询并读取数据
+ ShuffleRDD：shuffle操作生成的RDD
+ UnionRDD：
+ PartitionPruningRDD

**compute**：`compute()`是对RDD每个分区求值的函数，是RDD基类的一个抽象方法，RDD的每个子类都覆盖这个方法

## 创建RDD ##

有两种方式创建RDD：一是并行化driver program中存在的集合，一是引用外部存储系统中的数据集。

`SparkContext.parallelize()`用于创建并行集合。禁止使用`parallelize(Seq())`创建空RDD，使用emptyRDD表示不带分区的RDD，或者使用`parallelize(Seq[T]())`创建带有空分区的类型为T的RDD。

```scala
def parallelize[T](seq: Seq[T], numSlices: Int = defaultParallelism)(implicit arg0: ClassTag[T]): RDD[T]
```

Spark支持从Hadoop兼容的外部存储系统创建RDD，且支持文本文件、Sequence文件以及其他Hadoop InputFormat。

使用`SparkContext.textFile()`读取文本文件，每行对应结果RDD中的一个元素。参数path是文件路径，参数minPartitions是建议结果RDD的最小分区数，默认为defaultMinPartitions，而在SparkContext中defaultMinPartitions定义为`def defaultMinPartitions: Int = math.min(defaultParallelism, 2)`，表明其不会大于2。

```scala
def textFile(path: String, minPartitions: Int = defaultMinPartitions): RDD[String]
```

使用Spark读取文件时需要注意以下几点：

+ 如果使用本地文件系统路径，文件必须存在于worker节点的相同路径
+ Spark所有基于文件的输入方法支持目录、压缩文件和通配符
+ Spark默认为文件的每个HDFS块（默认为128M）创建一个分区，可以通过minPartitions设置更大的分区数目，分区数目不能比块数目小

使用`SparkContext.wholeTextFiles()`读取包含多个小文本文件的目录，结果RDD中的元素为`(filename, content)`的形式，分区由data locality决定，可能导致分区过少，可以通过minPartitions参数控制分区数目

```scala
def wholeTextFiles(path: String, minPartitions: Int = defaultMinPartitions): RDD[(String, String)]
```

使用`SparkContext.sequenceFile[K,V]()`读取Sequence文件，K和V分别是文件中键和值的类型，应该是Hadoop Writable接口的子类，比如`IntWritable`、`Text`。

```scala
def sequenceFile[K, V](path: String, minPartitions: Int = defaultMinPartitions): RDD[(K, V)]
def sequenceFile[K, V](path: String, keyClass: Class[K], valueClass: Class[V]): RDD[(K, V)]
def sequenceFile[K, V](path: String, keyClass: Class[K], valueClass: Class[V], minPartitions: Int): RDD[(K, V)]
```

## RDD操作 ##

RDD支持两种操作：transformation和action。RDD transformation从已经存在的RDD创建新的RDD，新RDD是惰性求值的，RDD action则向driver program返回结果或把结果写入外部系统。

### Transformation ###

通用RDD transformation：

|名称|描述|函数参数类型|
|---|---|---|
|`filter(func)`|返回满足条件的元素组成的RDD|`(T) => Boolean`|
|`map(func)`|对 RDD中的每个元素应用函数func，返回新元素组成的RDD|`(T) ⇒ U`|
|`mapPartitions(func)`|对RDD的每个分区应用函数func，返回新元素组成的RDD。如果在map过程中需要频繁创建额外的对象，那么mapPartitions效率比map高很多|`(Iterator[T]) => Iterator[U]`|
|`mapPartitionsWithIndex(func)`|与 mapPartitions类似，传入的参数多了一个分区的索引值。preservesPartitioning表示是否保留原partitioner，一般为false，除非RDD为键值对RDD并且func不修改键值|`(Int, Iterator[T]) => Iterator[U]`|
|`flatMap(func)`|对RDD中的每个元素应用func并将结果扁平化，返回新元素组成的RDD|`(T) => TraversableOnce[U]`|
|`pipe(command)`|调用shell命令处理RDD||
|`groupBy(func)`|根据给定的func对RDD中元素进行分组，返回一个键值对RDD，键由对RDD中元素应用f后返回的值组成，值为对应相同键的所有元素组成的序列，每组内元素是无序的|`(T) => K`|
|`glom()`|将类型为T的RDD的一个分区内的所有元素合并到一个数组中，返回一个Array[T]类型的RDD||
|`keyBy(func)`|返回一个键值对RDD，键为对元素应用函数func后的返回值|`(T) => K`|
|`sortBy(func)`|对RDD排序|`(T) ⇒ K`|
|`repartition(numPartitions)`|通过shuffle操作，重新设置RDD的分区数，可以增加或减少分区。如果只需减少分区，应使用coalesce来避免shuffle操作||
|`coalesce(numPartitions)`|减少RDD分区||
|`distinct()`|对RDD去重||
|`union(otherRDD)`|合并两个RDD||
|`cartesian(otherRDD)`|计算两个RDD的笛卡尔积，返回一个键值对RDD，其中键为这个RDD中的元素，值otherRDD中的元素||
|`intersection(otherRDD)`|计算两个RDD的交集||
|`subtract(otherRDD)`|计算两个RDD的差集，返回所有存在这个RDD中同时不在otherRDD中的元素组成的新RDD||
|`zip(otherRDD)`|对两个RDD执行zip操作，返回一个键值对RDD，其中键是这个RDD中的元素，值是otherRDD中对应位置处的元素，这两个RDD必须有相同的分区并且每个分区的元素数目也必须相同||
|`zipPartitions()`|||
|`zipWithIndex()`|对RDD中的元素和它们对应的索引执行zip操作||
|`zipWithUniqueId()`|对RDD中的元素和一个唯一ID执行zip操作||

键值对RDD transformation：

|名称|描述|函数参数签名|
|---|---|---|
|`keys()`|获取所有键||
|`values()`|获取所有值||
|`groupByKey()`|将键值对RDD按照键分组，返回`(K,Iterable[V])`类型的RDD||
|`reduceByKey(func)`|将键值对RDD按照键使用函数func进行归约|`(V, V) => V`|
|`aggregateByKey(zeroValue)(seqOp, combOp, [numPartitions])`|使用指定的函数及初始值 (类型默认零值)，先对每个分区的值按照键进行归约操作，然后对所有分区的结果再进行归约操作|seqOp：`(U, V) => U` combOp：`(U, U) => U`|
|`combineByKey()`|||
|`foldByKey()`|使用函数func和零值聚合每个键对应的值，零值可能多次加到结果上，但是不能改变结果值||
|`sortByKey()`|对键值对RDD根据键进行排序||
|`join(otherRDD)`|对两个键值对RDD进行join操作，结果RDD中元素类型为`(k, (v1, v2))`， v1、v2分别为两个RDD中k对应的值，这个操作将在集群中进行hashjoin||
|`leftOuterJoin(otherRDD)`|对两个键值对RDD进行left outer join操作||
|`rightOuterJoin(otherRDD)`|对两个键值对RDD进行right outer join操作||
|`fullOuterJoin(otherRDD)`|对两个键值对RDD进行full outer join操作||
|`cogroup()`|组合多个RDD中相同键对应的值，返回`(K, (Iterable[V1], Iterable[V2]))`类型的RDD||
|`mapValues(func)`|对键值对RDD中的值应用函数func|`(V) => U`|
|`flatMapValues(func)`|对键值对RDD中的值应用函数func，对结果数组进行扁平化后和键结合|`(V) => TraversableOnce[U]`|
|`repartitionAndSortWithinPartitions()`|||

### Action ###

## 其他 ##

### RDD持久化 ###

Spark的一个很重要的能力就是将数据持久化（或称为缓存），在多个操作间都可以访问这些持久化的数据。当持久化一个RDD时，每个节点会将本节点计算的数据块存储到内存，在该数据上的其他action操作将直接使用内存中的数据。这样会让以后的action操作计算速度加快。缓存保障迭代算法和交互使用快速进行的重要手段。

RDD可以使用`persisit()`或`cache()`进行持久化。数据将会在第一次action操作时进行计算，并在各个节点的内存中缓存。Spark缓存具有容错机制，如果一个缓存的RDD的某个分区丢失了，Spark将按照原来的计算过程，自动重新计算并进行缓存。

另外，每个持久化的RDD可以使用不同的存储级别进行缓存，比如，持久化到磁盘、已序列化的Java对象持久化到内存、跨节点复制、已off-heap的方式存储在
