# MapReduce #

MapReduce是一个可靠的、容错的分布式数据处理框架。将业务处理逻辑抽象到map和reduce方法中，和作业的控制逻辑分离。

一个MapReduce作业通常将输入数据集划分成独立的块，并由map任务并行的处理。框架对map的结果排序后传入reduce任务。作业的输入和输出一般都存储在文件系统中。框架负责调度、监控任务，并重新执行失败的任务。

数据模型：`(input)<k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3>(output)`

## 原理 ##

### reduce ###

MapReduce框架保证reducer的输入按键排序。

reduce分为三个阶段：shuffle、sort和reduce。

#### shuffle ####

shuffle是框架执行排序并把map输出传入reducer的过程，是MapReduce的核心。

![shuffle and sort](images/shuffleandsort.jpeg)

**map侧**：map并不是直接将输出写入磁盘，而是写入循环内存缓冲区（circular memory buffer），缓冲区默认100MB大小，可以通过`mapreduce.task.io.sort.mb`调整。当缓冲区的数据达到某个阈值（`mapreduce.map.sort.spill.percent`，默认为0.80）时，一个后台线程将开始将缓冲区中的数据写入磁盘（称为spill）。spill进行时map继续将输出写入缓冲区，但是当缓冲区写满时，map将阻塞直到spill完成。spill以轮询的方式将数据写入`mapreduce.cluster.local.dir`指定目录下的作业特定的子目录中。数据写入磁盘前，线程先根据数据将要发送到的reducer将它们分区。后台线程对每个分区中的数据按照键进行内存内排序，如果存在combiner函数，那么它将在排序的输出上运行。使用combiner函数将使map输出更紧凑，减少写入本地磁盘及发送给reducer的数据量。每次循环内存达到spill阈值时，都会创建一个新的spill文件，所以在map任务写出最后一条记录后，将会有多个spill文件。任务结束前，这些spill文件将会合并进一个单独的分区有序输出文件。`mapreduce.task.io.sort.factor`控制同时合并流的最大数目，默认是10。如果至少有三个spill文件（由`mapreduce.map.combine.minspills`设置），写入输出文件前将再次运行combiner。combiner需要满足运行多次且不影响最终结果。如果只有一个或两个spill文件，combiner减少map输出数据大小的效果比不上调用combiner的开销，所以不会再次运行combiner。最好在map输出写入磁盘前进行压缩，这样做将缩短写入磁盘的时间，节省磁盘空间，并减少发送到reducer的数据大小。默认输出不压缩，设置`mapreduce.map.output.compress`为true将启用压缩，使用的压缩库由`mapreduce.map.output.compress.codec`指定。输出文件的分区通过HTTP发送到reducer，`mapreduce.shuffle.max.threads`指定每台node manager发送分区的工作线程的最大数目。默认为0，表示线程最大数目为机器处理器数目的两倍。

**reducer侧**：map的输出文件位于运行map任务的机器的本地磁盘（map输出总是写入本地磁盘，而reduce输出可能不是），运行reduce任务的机器需要获取其将要处理的分区，分区中的数据来自集群上的多个map任务。map任务可能不同时结束，每个map任务结束后，reduce任务就开始复制它的输出。成为reduce任务的复制阶段。reduce任务有少量的复制线程来并行获取map输出。默认为五个线程，可以通过`mapreduce.reduce.parallelcopies`设置。map任务成功后，使用心跳机制通知application master。所以，对于给定任务，application master知道map输出和host之间的映射关系。reducer的一个线程周期性的向application master询问map输出host直到接收完所有数据。第一个reducer获取数据后host并不立即从磁盘删除map输出，因为reducer可能失败。相反，它们直到作业结束后被application master告知删除map输出时，才进行删除。map输出很小时将会被复制到reduce任务的JVM内存中（由`mapreduce.reduce.shuffle.input.buffer.percent`指定）；否则，它们将被复制到磁盘。当内存缓冲区到达阈值（`mapreduce.reduce.shuffle.merge.percent`）时，或map输出数目到达阈值时（`mapreduce.reduce.shuffle.merge.inmem.threshold`），内存缓存区中的数据将被合并并写入磁盘。指定combiner时，它将会在合并时执行来减少写入到磁盘的数据量。当磁盘上复制的数据增加时，一个后台线程将把它们合并成一个更大的有序文件，这将减少后面合并的时间。注意任何被压缩的map输出都要在内存中解压缩进而进行合并。当复制完所有map输出后，reduce任务进入到sort阶段（称为merger阶段更合适，因为sort发生在map侧），合并map输出，维持排序顺序。这将进行好几轮。比如，如果有50个map输出，并且合并因子是10（默认值，由`mapreduce.task.io.sort.factor`指定），将会进行五轮，每轮合并10个文件，最后生成5个中间文件。不再进行一轮来将5个文件合并成一个单独的排序文件，而是在最后一个阶段--reduce阶段--直接将数据传入reduce函数来减少一次写入磁盘的操作。最后的合并为内存内和磁盘上数据片段的混合。reduce阶段，对有序输出中的每个键调用一次reduce函数，这个阶段的输出直接写入输出文件系统，一般是HDFS。对于HDFS，由于node manager也是datanode，第一个块副本将被写入本地磁盘。

### map ###

map将输入键值对映射到中间键值对，中间键值对的类型不必和输入键值对的相同，一个输入键值对可以生零个活多个键值对。

### reduce ###

## API ##

```plantUML
@startuml
skinparam shadowing false

skinparam class {
    BorderColor #8FBC8F
    BackgroundColor White
    ArrowColor Gray
}

class Mapper <KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
    void setup(Mapper.Context context);
    void map(KEYIN key, VALUEIN value, Mapper.Context context);
    void cleanup(Mapper.Context context);
    void run(Mapper.Context context);
}

class Reducer <KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
    void setup(Reducer.Context context);
    void reduce(KEYIN key, Iterable<VALUEIN> values, Reducer.Context context);
    void cleanup(Reducer.Context context);
    void run(Reducer.Context context);
}

hide field
@enduml
```

Mapper：

+ `setup()`：任务开始时调用一次
+ `map()`：对输入分片（input spilt）中的每个键值对调用一次，默认为恒等函数
+ `cleanup()`：任务结束时调用一次
+ `run()`：对map处理进行更深入的控制

MapReduce框架内置了一些具有特定功能的Mapper：

+ `ChainMapper`：用于在单个Map任务中使用多个Mapper类。这些Mapper类将以管道形式调用，即第一个的输出成为第二个的输入，直到最后一个Mapper，最后一个Mapper的输出作为整个任务的输出。和`ChainReducer`联用实现链式（`[MAP+ / REDUCE MAP*]`）的MapReduce作业，大大减少磁盘IO
  * `addMapper()`方法用于添加Mapper类
+ `FieldSelectionMapper`：执行字段选取
+ `InverseMapper`：交换键值
+ `MultithreadedMapper`：Mapper的多线程实现，当Map处理每条任务都很耗时，可以考虑`MultithreadedMapper`，对于IO密集型任务可能会带来性能提升
+ `RegexMapper`：提取和正则表达式匹配的文本
+ `TokenCounterMapper`：使用StringTokenizer拆分值中的单词并输出`<word, one>`对
+ `ValueAggregatorMapper`：实现了通用聚集Mapper
+ `WrappedMapper`：用于自定义Context

Reducer:

+ `setup()`：任务开始时调用一次
+ `reduce()`：对每个键调用一次，默认为恒等函数
+ `cleanup()`：任务结束时调用一次
+ `run()`：控制reduce任务如何执行

MapReduce框架内置了一些具有特定功能的Reducer：

+ `ChainReducer`：用于在单个Reduce任务中Reducer后链式调用多个Mapper
+ `FieldSelectionReducer`：进行字段选取
+ `IntSumReducer`：进行整型归约
+ `LongSumReducer`：进行长整型归约
+ `ValueAggratorCombiner`：实现通用聚集Combiner
+ `ValueAggratorReducer`：实现通用聚集Reducer
+ `WrappedReducer`：用于自定义Context

```java
import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
public class WordCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }
  }

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```
