## spark bug

1. exceeding memory limits

* spark作业
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
    Job aborted due to stage failure: Task 23 in stage 3.0 failed 4 times, most recent failure: Lost task 23.3 in stage 3.0 (TID 584, zjy-hadoop-prc-st3768.bj, executor 166): ExecutorLostFailure (executor 166 exited caused by one of the running tasks) Reason: Container killed by YARN for exceeding memory limits. 8.12 GB of 8 GB physical memory used. Consider boosting spark.yarn.executor.jvmMemoryOverhead.
    Driver stacktrace:
    ```
* 查看对应的executor log
    ```
    ExecutorLostFailure (executor 248 exited caused by one of the running tasks) Reason: Container killed by YARN for exceeding memory limits. 8.17 GB of 8 GB physical memory used. Consider boosting spark.yarn.executor.jvmMemoryOverhead.
    ```
* 资源使用情况
    ```
        指标	当前	建议	描述
    堆峰值内存	5888m	spark.executor.memory=5888m	executor堆内存当前配置:6g
    堆外峰值内存	128m	spark.yarn.executor.memoryOverhead=512m	executor堆外内存当前配置:2048MB
    堆内存最大的ExecutorId	295		
    堆外内存最大的ExecutorId	30		
    ```
* 分析
  * executor内存不足
  * executor.memoryOverhead实际使用128m,物理内存实际利用很低，但虚拟内存却很高。
* 解决办法
  * 增加executor.memoryOverhead
  * 该问题的原因是因为OS层面虚拟内存分配导致，物理内存没有占用多少，但检查虚拟内存的时候却发现OOM，因此可以通过关闭虚拟内存检查来解决该问题，yarn.nodemanager.vmem-check-enabled=false

* 知识点
  * https://blog.csdn.net/lmb09122508/article/details/84822906

2. Executor heartbeat timed out

* spark Job
  * 任务
    * 根据用户点击数据计算item相似
  * 数据
    * clickSeq 大小：<10g，数据格式：sessionId  uid gid time
  * 作业代码
    ```java
    // 读取原数据，组织成Rdd((uid,list((gid,time)...)))的形式
    val clickMatrixRdd = sc.textFile(clickSeq).map{line=>{
      val cols = line.split("\t")
      if(cols.length==4 && !cols(1).equals("0")){
        (cols(1),cols(2),cols(3).toDouble)
      }else{
        null
      }
    }}.filter(item=> item!=null && itemStatusMap.getOrElse(item._2,0)==1)
      .map{case (uid,gid,time)=>{
        ((uid,List((gid,time))))
      }}.reduceByKey(_++_)
    ```
* stages报错
    ```
    Job aborted due to stage failure: Task 8 in stage 2.0 failed 4 times, most recent failure: Lost task 8.3 in stage 2.0 (TID 83, zjy-hadoop-prc-st1685.bj, executor 194): ExecutorLostFailure (executor 194 exited caused by one of the running tasks) Reason: Executor heartbeat timed out after 127874 ms +details
    Job aborted due to stage failure: Task 8 in stage 2.0 failed 4 times, most recent failure: Lost task 8.3 in stage 2.0 (TID 83, zjy-hadoop-prc-st1685.bj, executor 194): ExecutorLostFailure (executor 194 exited caused by one of the running tasks) Reason: Executor heartbeat timed out after 127874 ms
    Driver stacktrace:
    ```
* 查看对应的executor log
  * 无Error字段
  * gc log显示gc没有足够的空间
    ```
    2020-06-28T19:42:40.544+0800: 2657.475: [GC (Allocation Failure) [PSYoungGen: 3058174K->436731K(3058176K)] 3741310K->1642553K(10049024K), 1.5991417 secs] [Times: user=4.97 sys=1.41, real=1.59 secs] 
    ```
* 分析
  * Eden没有可用空间分配给对象
  * 作业中需要对相同uid的List((gid,time))进行合并，会产生大量的中间List
* 解决办法
  * 尝试过扩大executor memory但依然不行
  * 尝试增加executor and network timeout，但因权限问题无法配置
        -conf spark.network.timeout 10000000 
        --conf spark.executor.heartbeatInterval=10000000 
  * 修改代码逻辑
      * uid数量庞大，每个uid对应的gid也很多，在reduceBykey的过程中势必会有大量的_++_操作，这里可以采取分解合并的策略，即先根据（uid，time）进行聚合，再根据uid进行聚合，将原来一步的操作拆分为两步进行 
        ```java
        val clickMatrixRdd = sc.textFile(clickSeq).map{line=>{
        val cols = line.split("\t")
        if(cols.length==4 && !cols(1).equals("0")){
            (cols(1),cols(2),cols(3).toDouble)
        }else{
            null
        }
        }}.filter(item=> item!=null && itemStatusMap.getOrElse(item._2,0)==1)
        .map{case (uid,gid,time)=>{
            val sessionId = time / 1000 / 60 / 60 /24
            ((uid,sessionId),List((gid,time)))
        }}.reduceByKey(_++_)
        .map{case ((uid,sessionId),list)=>{
            (uid,list)
        }}.reduceByKey(_++_)
        ```
* 知识点
  * https://stackoverflow.com/questions/28342736/java-gc-allocation-failure
  * https://stackoverflow.com/questions/54036028/spark-executor-heartbeat-timed-out-after-x-ms
  * https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html


3. llegalArgumentException: Size exceeds Integer.MAX_VALUE
  * spark Job
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

* 知识点
  * https://stackoverflow.com/questions/42247630/sql-query-in-spark-scala-size-exceeds-integer-max-value