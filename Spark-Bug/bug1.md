#### spark作业
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

#### 知识点及参考
* 知识点

* 参考
    * https://stackoverflow.com/questions/28342736/java-gc-allocation-failure
    * https://stackoverflow.com/questions/54036028/spark-executor-heartbeat-timed-out-after-x-ms
    * https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html
