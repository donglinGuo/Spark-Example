#### spark-submit script
* 位于spark安装目录bin文件夹下的spark-submit脚本用于将应用程序提交到集群/本地，对所有的集群管理起来说它是加载spark程序的统一入口
* spark应用需要将程序和对应的依赖一块打包成jar包，然后分发给集群。对于spark和hadoop只需要定义其依赖，但并不需要绑定打包，因为集群管理器会在运行时提供相应的支持;对于python程序，可以在spark-submit中通过配置--py-files参数将依赖和你的程序一块分发，--py-files参数后面可以跟.py, .zip或则.egg文件。
* 打包成jar包后就可以通过spark-submit脚本提交应用程序，该脚本负责配置spark相关的classpath和依赖，并支持不同的集群管理器和部署模式。
* spark-submit脚本常用的选项
    ```
    ./bin/spark-submit \
    --class <main-class> \
    --master <master-url> \
    --deploy-mode <deploy-mode> \
    --conf <key>=<value> \
    ... # other options
    <application-jar> \
    [application-arguments]
    ```
    --class: spark程序的入口 \
    --master: 集群的master url \
    --deploy-mode: 是将driver节点放在集群中（cluster）还是本地（client） \
    --conf： 以key=value的形式配置spark的属性，如果value中包含空格则需要使用引号阔起来--conf "key=value" \
    application-jar： jar包的路径，该路径需要对集群可见，如hdfs路径，或则所有节点上的file://路径 \
    application-arguments： 传递给主函数main方法的参数 \

* 部署模式deploy-mode，当提交程序的机器与集群物理位置比较近时可以选用client方式部署，这样driver节点和集群节点通信不会影响程序执行性能，如果物理位置比较远，需要选用cluster方式部署，既在集群中选择一个节点作为driver。
* 对于python程序使用.py文件替换application-jar包，py-spark会通过--py-files参数查找对应的依赖文件
* master用于选择特定的集群管理器，master URL可用的格式如下
    * local 本地单线程
    * local[k] 本地k个线程
    * local[k,F] 本地K个线程，允许最大失败次数F
    * local[*] 本地尽可能多的线程（logical cores决定)
    * local[*,F]
    * spark://HOST:PORT spark的standalone cluster master，port默认为7077
    * spark://HOST1:PORT1,HOST2:PORT2 通过zookeeper连接给定的spark standalone cluster
    * mesos://HOST:PORT 或则 mesos://zk://...(部署模式只能是cluster) 默认端口为5050
    * yarn 链接yarn集群，部署模式可以是client也可以是cluster。集群的路径通过HADOOP_CONF_DIR或则YARN_CONF_DIR变量获取
    * k8s://HOST:PORT 使用cluster模式连接Kubernetes集群
* saprk在提交程序时会读取安装目录下的默认配置文件conf/saprk-defaults.conf。不同的配置方式优先级不同，sparkconf>spark-submit script>spark-default.conf。如果不清楚配置选项是在哪里配置的，可以在spark-submit的时候使用--verbose参数，打印相关debugging信息
* 如果spark程序额外依赖其它jar包，可以通过--jars配置其路径（多个路径使用","分隔开），--jars配置的jar包和程序jar包会被自动的分发到集群中

















http://spark.apache.org/docs/latest/submitting-applications.html