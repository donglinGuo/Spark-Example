#### spark作业
* 任务
    * 处理用户日志数据，根据某一字段做flatMap然后聚合
* 数据
    * 回跑过去30天日志并union >120G 
* 作业代码
    ```java
    val type3ScoreRdd = actionRdd.map{case (uid,pid,action,timestamp,source_id,spm,actionCount)=>{
     ((uid,pid,timestamp),actionCount)
   }}.flatMap{case ((userId,productId,hour),count)=>{
     productIdtypeMap.getOrElse(productId,(Set(-1),Set(-1)))._1.map{case type3=>{
       (type3,(count._1,count._2,count._3,count._4))
     }}
   }}.reduceByKey((a, b) => (a._1 + b._1, a._2 + b._2, a._3 + b._3, a._4 + b._4))
    ```
* stages报错
    ```
    Job aborted due to stage failure: Task 23 in stage 3.0 failed 4 times, most recent failure: Lost task 23.3 in stage 3.0 (TID 633, zjy-hadoop-prc-st4203.bj, executor 261): java.lang.IllegalArgumentException: Size exceeds Integer.MAX_VALUE
    ```
* 查看对应的executor log
    ```
    java.lang.IllegalArgumentException: Size exceeds Integer.MAX_VALUE
	at sun.nio.ch.FileChannelImpl.map(FileChannelImpl.java:863)
	at org.apache.spark.storage.DiskStore$$anonfun$getBytes$2.apply(DiskStore.scala:107)
	at org.apache.spark.storage.DiskStore$$anonfun$getBytes$2.apply(DiskStore.scala:93)
	at org.apache.spark.util.Utils$.tryWithSafeFinally(Utils.scala:1314)
	at org.apache.spark.storage.DiskStore.getBytes(DiskStore.scala:109)
	at org.apache.spark.storage.BlockManager.getLocalValues(BlockManager.scala:464)
	at org.apache.spark.storage.BlockManager.getOrElseUpdate(BlockManager.scala:748)
	at org.apache.spark.rdd.RDD.getOrCompute(RDD.scala:334)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:285)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:323)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:287)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:323)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:287)
	at org.apache.spark.scheduler.ShuffleMapTask.runTask(ShuffleMapTask.scala:96)
	at org.apache.spark.scheduler.ShuffleMapTask.runTask(ShuffleMapTask.scala:53)
	at org.apache.spark.scheduler.Task.run(Task.scala:100)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:340)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
    ```
* 资源使用情况
    ```
        指标	当前	建议	描述
    堆峰值内存	5888m	spark.executor.memory=5888m	executor堆内存当前配置:6g
    堆外峰值内存	128m	spark.yarn.executor.memoryOverhead=512m	executor堆外内存当前配置:2048MB
    堆内存最大的ExecutorId	295		
    堆外内存最大的ExecutorId	30		
    ```


*   No Spark shuffle block can be larger than 2GB (Integer.MAX_VALUE bytes) so you need more / smaller partitions.
    You should adjust spark.default.parallelism and spark.sql.shuffle.partitions (default 200) such that the number of partitions can accommodate your data without reaching the 2GB limit (you could try aiming for 256MB / partition so for 200GB you get 800 partitions). Thousands of partitions is very common so don't be afraid to repartition to 1000 as suggested.
* 解决办法
    * 使用repartition增加partitions

#### 知识点及参考
* 知识点

* 参考
    * https://stackoverflow.com/questions/42247630/sql-query-in-spark-scala-size-exceeds-integer-max-value