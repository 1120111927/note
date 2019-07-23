# Parquet #

parquet是一种支持列式存储的文件格式，基于Dremel的数据模型和算法实现。

parquet是语言无关的，而且不与任何一种数据处理框架绑定在一起，适配多种语言和组件，与parquet协作的组件如下：

+ 查询引擎：hive、impala、presto
+ 计算框架：MapReduce、Spark
+ 数据模型：Avro、Thrift、Protocol Buffers

数据从内存到Parquet文件或者反过来的过程主要由以下三个部分组成：

+ 存储格式（storage format）：parquet-format项目定义了Parquet内部的数据类型、存储格式等
+ 对象模型转换器（object model converters）：由parquet-mr项目来实现，主要完成外部对象模型与Parquet内部数据类型的映射
+ 对象模型（object model）：指内存中的数据表示，如Avro、Protocol Buffers

## 数据模型 ##

```code
message <Record> {
    <repetition> <primitiveType> <name>;
    ...
    <repetition> group <name> {
        <repetition> <primitiveType> <name>;
        ...
    }
}
```

每个schema的结构是这样的：根称为message，message包含多个field。每个field包含三个属性：repetition, type, name。repetition可以是以下三种：required（出现1次），optional（出现0次或者1次），repeated（出现0次或者多次）。type可以是一个group或者一个primitive类型。

Parquet使用repeated field和group来表示Map, List, Set等复杂类型。List和Set可以被表示成一个repeated field，Map可以表示成一个包含有key-value对的repeated field，而且key是required的。

基本类型有：

+ BOOLEAN
+ INT32
+ INT64
+ INT96
+ FLOAT
+ DOUBLE
+ BYTE_ARRAY

### Striping/Assembly算法 ###

使用Repetition Level、Definition Level和value来序列化和反序列化嵌套数据类型。在parquet中只需定义和存储schema的叶子节点所在列的Repetition Level和Definition Level。

Repetition Level用于处理repeated类型的field，在写入时该值等于它和前面的值从哪一层节点开始是不共享的，在读取的时候根据该值可以推导出哪一层上需要创建一个新的节点。

Definition Level用于处理repeated和optional类型的field。definition level的值仅仅对于空值是有效的，表示在该值的路径上第几层开始是未定义的，对于非空的值它是没有意义的。

详细参考论文[Dremel](https://ai.google/research/pubs/pub36632)

## 文件存储格式 ##

```code
4-byte magic number "PAR1"
<Column 1 Chunk 1 + Column Metadata>
<Column 2 Chunk 1 + Column Metadata>
...
<Column N Chunk 1 + Column Metadata>
<Column 1 Chunk 2 + Column Metadata>
<Column 2 Chunk 2 + Column Metadata>
...
<Column N Chunk 2 + Column Metadata>
...
<Column 1 Chunk M + Column Metadata>
<Column 2 Chunk M + Column Metadata>
...
<Column N Chunk M + Column Metadata>
File Metadata
4-byte length in bytes of file metadata
4-byte magic number "PAR1"
```

术语：

+ HDFS块（Block）：HDFS上最小的存储单位，HDFS会把一个Block存储在一个本地文件并且维护分散在不同机器上的多个副本，通常情况下一个Block的大小为256M、512M等
+ HDFS文件（File）：一个HDFS文件，包括数据和元数据，数据分散存储在多个Block中
+ 行组（Row Group）：按照行将数据物理上划分为多个单元，每一个行组包含一定的行数，在一个HDFS文件中至少存储一个行组，parquet读写的时候会将整个行组缓存在内存中
+ 列块（Column Chunk）：在一个行组中每一列保存在一个列块中，行组中的所有列连续的存储在这个行组文件中。一个列块中的值都是相同类型的，不同的列块可能使用不同的算法进行压缩
+ 页（Page）：每一个列块划分为多个页，一个页是最小的编码的单位，在同一个列块的不同页可能使用不同的编码方式

parquet文件以二进制方式存储，文件中包括该文件的数据和元数，是自解析的。

parquet文件中所有数据被水平切分成Row Group，一个Row Group包含这个Row Group对应的区间内所有列的Column Chunk，一个Column Chunk负责存储某一列的数据，包括这一列的Repetition Level，Definition Level和value。

一个parquet文件中可以存储多个行组，文件的首位都是该文件的Magic Code，用于校验它是否是一个Parquet文件，Footer length记录了文件元数据的大小，通过该值和文件长度可以计算出元数据的偏移量，文件的元数据中包括每一个行组的元数据信息和该文件存储数据的Schema信息。除了文件中每一个行组的元数据，每一页的开始都会存储该页的元数据，在parquet中，有三种类型的页：数据页、字典页和索引页。数据页用于存储当前行组中该列的值，字典页存储该列值的编码字典，每一个列块中最多包含一个字典页，索引页用来存储当前行组下该列的索引。

有三种元数据：文件元数据，列元数据，页头元数据。

在执行MapReduce任务的时候可能存在多个Mapper任务的输入是同一个parquet文件的情况，每一个Mapper通过InputSplit标识处理的文件范围，如果多个InputSplit跨越了一个Row Group，parquet能够保证一个Row Group只会被一个Mapper任务处理。

通常情况下，在存储Parquet数据的时候会按照Block大小设置行组的大小，由于一般情况下每一个Mapper任务处理数据的最小单位是一个Block，这样可以把每一个行组由一个Mapper任务处理，增大任务执行并行度。

并行处理的基本单元：

+ MapReduce：File/Row Group
+ IO：列块
+ Encoding/Compression：Page

## 优势 ##

和行式存储相比，列式存储的性能优势主要体现在两个方面：一是节省存储，相同类型的数据更容易针对不同类型的列使用高效的编码和压缩方式，提高压缩比；二是提升查询性能，由于映射下推和谓词下推的使用，可以减少一大部分不必要的数据扫描，减少I/O操作。

| 压缩算法 | 使用场景 |
| --- | --- |
| Run Length Encoding | 重复数据 |
| Delta Encoding | 有序数据集 |
| Dictionary Encoding | 小规模的数据集合 |
| Prefix Encoding | Delta Encoding for strings |

映射下推（Project PushDown）：是指在获取表中原始数据时只需要扫描查询中需要的列，由于每一列的所有值都是连续存储的，所以分区取出每一列的所有值就可以实现TableScan算子，而避免扫描整个表文件内容。

谓词下推（Predicate PushDown）：是指将一些过滤条件尽可能的在最底层执行，可以减少每一层交互的数据量，从而提升性能。无论是行式存储还是列式存储，都可以在读取一条记录后执行过滤条件以判断该记录是否需要返回给调用者，parquet做了进一步优化，对每一个Row Group中的每一个Column Chunk在存储的时候都计算对应的统计信息，包括该Column Chunk的最大值、最小值和空值个数。通过这些统计值和该列的过滤条件可以判断该Row Group是否需要扫描。
