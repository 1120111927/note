# Hive 参数 #

1. 设置任务名：
    `set mapreduce.job.name=<jobName>`
2. 合并输入：

    ```hive
    set mapreduce.input.fileinputformat.split.maxsize=300000000;
    set mapreduce.input.fileinputformat.split.minsize=100000000;
    set mapreduce.input.fileinputformat.split.minsize.per.node=100000000;
    set mapreduce.input.fileinputformat.split.minsize.per.rack=100000000;
    set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
    set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;  -- 不进行小文件合并
    ```

3. 合并输出：

    ```hive
    set hive.merge.mapfiles=true; -- 在Map-only的任务结束时合并小文件
    set hive.merge.mapredfiles=true; -- 在MapReduce任务结束时合并小文件
    set hive.merge.size.per.task=256*1000*1000; -- 合并文件的大小
    set hive.merge.smallfiles.avgsize=16000000; -- 当输出文件的平均大小小于该值时，启动一个独立的MapReduce任务进行文件合并
    ```

4. reduce设置：

    ```hive
    set hive.exec.reducers.bytes.per.reducer=300000000;
    set hive.exec.reducers.max=300;
    set mapred.reduce.tasks=10; -- 固定reduce大小
    ```

5. mapjoin参数设置

    ```hive
    set hive.auto.convert.join=false; -- 是否开启mapjoin
    set hive.auto.convert.join.noconditionaltask=true; -- 是否将多个mapjoin合并成一个
    set hive.auto.convert.join.noconditionaltask.size=1000000; -- 多个mapjoin合并后的大小（阈值）
    ```

6. map端聚合

    ```hive
    set hive.map.aggr=true;
    ```

7. mapreduce的物理内存、虚拟内存

    ```hive
    -- 物理内存必须在允许范围内
    -- yarn.scheduler.maximum-allocation-mb=8192;
    -- yarn.scheduler.minimum-allocation-mb=1024;
    set mapreduce.map.memory.mb=4096;
    set mapreduce.reduce.memory.mb=4096;
    -- 堆内存大小要小于物理内存，一般设置为80%
    set mapreduce.map.java.opts=-Xmx1638m;
    set mapreduce.reduce.java.opts=-Xmx3278m;
    ```

8. 动态分区

    ```hive
    set hive.exec.dynamic.partition=true;
    set hive.exec.dynamic.partition.mode=nonstrict;
    ```

9. shuffle端内存溢出OOM

    ```hive
    set mapreduce.reduce.shuffle.memory.limit.percent=0.10;
    ```

10. map段谓词下推：

    ```hive
    set hvie.optimize.ppd=true;
    ```

11. 并行执行：

    ```hive
    set hive.exec.parallel=true;
    set hive.exec.parallel.thread.number=16;
    ```

12. reduce申请资源时机：

    ```hive
    set mapreduce.job.reduce.slowstart.completedmaps=0.05;  -- 控制当map任务执行到哪个比例的时候就可以开始为reduce task申请资源
    ```

    `mapreduce.job.reduce.slowstart.completedmaps`设置过低时，reduce就会过早申请资源，造成浪费；设置过高时，reduce就会过晚申请资源，不能充分利用资源。如果map数量比较多，一般建议提前开始为reduce申请资源

13. hive map、reduce任务个数：

    ```code
    map数：splitSize=Math.max(minSize, Math.min(maxSize, blockSize))
    reduce数：reducers=Math.min(maxReducers, totalInputFileSize/bytesPerReducer)
    ```

14. 数据倾斜：

    ```hive
    set hive.groupby.skewindata=true; -- 两轮mapreduce，先不按GroupBy字段分发，随机分发做一次聚合，然后再启动一次，对上次聚合的数据按GroupBy字段分发再算结果
    ```
