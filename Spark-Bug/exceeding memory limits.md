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

#### 知识点及参考
* 知识点

* 参考
    * https://blog.csdn.net/lmb09122508/article/details/84822906
