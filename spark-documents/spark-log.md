## Spark-Log

#### sparkUI

* 获取作业的sparkUI链接

* spark-ui
  * spark-ui中包括job,stages,storage,environment,executors
  * job，显示正在执行，已完成和正在执行的job,点击进入job链接可查看job详细信息
    * Job DAG依赖图
    * 完成/执行中/等待/失败的Stage
    * Job失败原因
  * stage，显示Stage的详细信息,
    * 包括执行时间,输入/shuffle/输出数据量和记录个数.
    * DAG依赖图
    * 完成/执行中/等待/失败的Task，Task失败原因, 相应Executor链接
  * storage，显示作业内存/磁盘使用情况, 当RDD有cache操作时候, 相应信息在显示在这个页面。
  * environment, 显示作业Spark/Hadoop/Java等版本以及Spark配置信息
  * executors, 显示作业Driver和Executor列表和日志

* job-log，Spark作业日志主要有四个类型
  * GC日志，显示driver和executor的GC情况
  * sparklog,运行日志，主要是Spark框架内的日志以及用户使用log接口的输出。
  * pmap日志，driver和executor的pmap和jstack，定时获取并通过RSS(Resident Set Size)指标来查看进程占用的物理内存情况。
  * stderr，系统错误日志，如果程序启动时出现异常可以查看该日志
  * launch-container-log,主要是挡前executor所在的container下的文件及executor环境变量等

* Driver报错
  * 如果spark任务失败，但在job页面没有提示错误信息，可以在executor tab页面查看driver日志
    * 查找"User class threw exception"
    * 查找"OutOfMemoryError"
    * 其它
* Executor报错
  * 如果spark任务失败，job页面有报错信息，则一般为executor报错。
    * 定位错误信息，查看出现错误的stage和task
    * 在stages tab定位task，确定错误信息
    * 查找具体执行的executor，确定错误信息