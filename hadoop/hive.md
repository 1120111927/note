# Hive #

## Map任务个数 ##

相关参数：

+ mapreduce.input.fileinputformat.split.minsize：map任务输入分片的最小字节数，默认为1
+ mapreduce.input.fileinputformat.split.maxsize：map任务输入分片的最大字节数，默认为Long.MAX_VALUE
+ dfs.blocksize：hdfs文件块的字节数，默认为128M

计算公式：

    SplitSize = Math.max(minsize, Math.min(maxsize, blocksize))
    MapTaskNum = sum(ceil(InputFileSize / SplitSize) for every InputFile)

## Reduce任务数 ##

相关参数：

+ hive.exec.reducers.bytes.per.reducer：每个reducer处理数据的字节数
+ hive.exec.reducers.max：reduce任务数目的最大值
+ mapreduce.job.reduces：设置reduce数目，默认值为-1，表示hive根据根据输入大小计算reduce数目，对于Hadoop该值默认为1

当mapreduce.job.reduces大于0时，reduce任务数即mapreduce.job.reduces，否则根据公式

    ReduceTaskNum = Math.max(1, Math.min(hive.exec.reducers.max, totalInputSize / bytesPerReducer))

计算reduce任务数。
