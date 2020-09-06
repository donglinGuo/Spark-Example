## Spark RDD Partition

#### 什么是Partition

在Spark-overview中提到RDD是Spark的基本数据结构，是一个对象集合，为了处理海量数据需要将RDD拆分成不同的Partition，不同的Partition在切分出的Stage内并行处理，并行处理的task数量和partition数量一致。因为集群中同一Job的tasks被分配到