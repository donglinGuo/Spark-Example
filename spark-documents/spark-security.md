## Spark的安全机制
#### Note
* spark的安全机制默认是关闭的。
* saprk支持不同的部署类型，不同的部署类型支持的安全级别不同。
* 不同场景（安全事件）的安全机制不同。

## 不同场景的安全机制
#### Spark RPC（进程之间的通信协议）
1. 通过共享密钥对RPC通道进行验证
    * 通过设置spark.authenticate参数，决定是否进行认证。
2. 共享密钥的产生与派发和部署类型相关
    * YARN和local类型下，spakr会自动生成并分发共享密钥，一个应用使用一个密钥。
    * 对于资源管理程序，spark会在每个节点上配置spark.authenticate.secret机制，该机制是被每个程序和守护进程共享的。
 
#### Kerberos
1. spark使用Kerberos对提交spark程序的环境进行验证。大多数情况下，Spark在认证Kerberos支持的服务时，会依赖于当前登录用户的证书，这些证书可以通过像kinit之类的工具登录到配置的KDC获取。
2. 当spark与基于hadoop的服务进行通信时，需要获取委派令牌，这样非本地程序才能够进行身份验证，spark支持对HDFS及其它Hadoop文件系统（Hive，HBase）的支持
3. 如果HBase在应用程序的classpath里，并且HBase开启了使用Kerberos进行验证(hbase.security.authentication=kerberos），则应用程序会自动获取HBase的令牌。
4. 如果Hive在应用程序的classpath里，并且Hive关于远程元存储服务的配置URIs不为空（hive.metastore.uris is not empty）
5. 如果应用需要和其它hadoop文件系统交互，则他们的URIs需要在Spark加载的时候显式提供，这通过在spark.kerberos.access.hadoopFileSystems属性中表明来完成
6. Spark还支持使用java服务机制（java.util.ServiceLoader）自定义的委派令牌提供程序，通过将hadoop令牌提供程序的名称列在jar包的META-INF/services路径下的文件里实现org.apache.spark.security.HadoopDelegationTokenProvider可用。
7. 委派令牌只在YARN和Mesos模式下可用
8. 如果应用程序执行时间过长超出委派令牌的有效时间可能会影响应用的执行，这种情况通常会发生在YARN和Kubernetes（client和modes模式）以及Mesos的client模式
    * spark支持为应用自动创建新的令牌，有两种方式实现
        * 使用Keytab：通过向spark提供principal和keytab（例如在spark-submit时配置--principal和--keytab参数）应用程序会维持一个有效的kerberos登录，该登录可以无限期的检索委派令牌。需要注意的是在集群模式下使用keytab时它会被复制到运行spark driver程序的机器上，对于YARN，会使用HDFS作为keytab的暂存区，因此建议对YARN和HDFS加密。
        * 使用ticket cache:通过设置spark配置参数spark.kerberos.renewal.credentials为ccache，来使用本地Kerberos ticket cache进行验证，spark会保持ticket更新，但当ticket过期时需要重新获取最新版的（例如通过运行kinit）;由用户维护并更新spark使用的ticket cache;通过设置环境变量KRB5CCNAME来设置ticket cache的位置。

        
        

http://spark.apache.org/docs/latest/security.html