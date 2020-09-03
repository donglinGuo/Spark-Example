
* 什么是Spark

  * spark是大规模分布式数据的计算引擎
  * spark支持Resilient Distributed Datasets (RDD) 
  * spark支持cyclic data flow 和 in-memory computing
  * 在内存中相较于Hadoop MapReduce速度快100倍，在磁盘上快10倍
  * 加速策略
    * DAG
    * Cache
  * 多种语言支持



* RDD
  * RDDs can be roughly viewed as partitioned, locality aware  distributed vectors
  * Goal: 
    * support wide array of operators and let users compose them arbitrarily
    * not modify scheduler for each one
    * capture dependencies generically
  * Interface 
    * Set of partitions (“splits”)
    * List of dependencies on parent RDDs
    * Function to compute a partition given parents
    * Optional: preferred locations
    * Optional: partitioning info partitioner


* RDD
  RDD是spark中基本的数据结构，它是一个只读（不可变）的对象集合，该集合存储在内存或磁盘中







https://www.tutorialspoint.com/apache_spark/apache_spark_rdd.htm

https://intellipaat.com/blog/tutorial/spark-tutorial/programming-with-rdds/

https://blog.csdn.net/u011564172/article/details/53611109

https://zhuanlan.zhihu.com/p/97777405