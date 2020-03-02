# UDAF

Spark 提供了两种聚合函数接口：

+ `UserDefinedAggregateFunction`：用于实现无类型的用户自定义聚合函数

+ `Aggregator`：用于实现类型安全的用户自定义函数

## 实现无类型的用户自定义聚合函数

### 实现`UserDefinedAggregateFucntion`

`UserDefinedAggregateFunction`在文件`org.apache.spark.sql.expressions.udaf`中定义：

```scala
abstract class UserDefinedAggregateFunction extends Serializable {

  def inputSchema: StructType  // 表示聚合函数输入参数的数据类型
  def bufferSchema: StructType  // 表示聚合缓冲区中值的数据类型
  def dataType: DataType  // 表示聚合函数返回值的数据类型
  def deterministic: Boolean  // 表示该聚合函数是否是确定性的

  def initialize(buffer: MutableAggregationBuffer): Unit  // 初始化聚合缓冲区，`merge(intitalBuffer, initialBuffer)`应该等于`initialBuffer`
  def update(buffer: MutableAggregationBuffer, input: Row): Unit  // 使用`input`中的输入数据更新指定的聚合缓冲区`buffer`，对每个输入行调用一次
  def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit  // 合并两个聚合缓冲区，并且把结果存储回`buffer1`，当合并两个部分聚合的数据时调用
  def evaluate(buffer: Row): Any  // 根据给定的聚合缓冲区`buffer`计算最终结果

  // 使用给定的Column作为输入参数生成一个Column
  def apply(exprs: Column*): Column = {
    val aggregateExpression =
      AggregateExpression(
        ScalaUDAF(exprs.map(_.expr), this),
        Complete,
        isDistinct = false)
    Column(aggregateExpression)
  }
  // 使用给定Column的不同值作为输入参数生成一个Column
  def distinct(exprs: Column*): Column = {
    val aggregateExpression =
      AggregateExpression(
        ScalaUDAF(exprs.map(_.expr), this),
        Complete,
        isDistinct = true)
    Column(aggregateExpression)
  }
}

// 表示可变聚合缓冲区
abstract class MutableAggregationBuffer extends Row {
  def update(i: Int, value: Any): Unit  // 更新缓冲区第i个值
}
```

### 注册UDAF

```scala
  spark.udf.register(<function_name>, new <UDAFClass>)
```

### 使用UDAF

```sql
select <function_name>(column[, column...]) from <table> [group by column[, column...]]
```

```scala
val udaf = new <UDAFClass>

dataframe.groupBy(column...).agg(udaf(column...).as(column))

dataframe.groupBy(column...).agg(expr("udaf(column) as column"))
```