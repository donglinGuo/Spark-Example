## spark overview

#### 大规模数据并行处理

* 谷歌面向大规模数据处理的并行计算模型和方法
    为了解决搜索引擎中大规模网页数据的并行化处理，google提出并在此基础上发展出了Google File System、Mapreduce、Bigtable

* Yahoo资助的Hadoop按照Google公开的三篇论文的开源Java实现:Hadoop mapreduce对应Google Mapreduce, Hadoop Distributed File System (HDFS)对应Google File System, Hbase对应Bigtable

* spark

#### mapreduce
首先大数据涉及两个方面：分布式存储系统和分布式计算框架。前者的理论基础是GFS。后者的理论基础为MapReduce。MapReduce框架有两个步骤（MapReduce 框架其实包含5 个步骤：Map、Sort、Combine、Shuffle 以及Reduce。这5 个步骤中最重要的就是Map 和Reduce。这也是和Spark 最相关的两步，因此这里只讨论这两个步骤）：一个是 Map，另一个是 Reduce。

Map 步骤是在不同机器上独立且同步运行的，它的主要目的是将数据转换为 key-value 的形式；而 Reduce 步骤是做聚合运算，它也是在不同机器上独立且同步运行的。Map 和 Reduce 中间夹杂着一步数据移动，也就是 shuffle，这步操作会涉及数量巨大的网络传输（network I/O），需要耗费大量的时间。 由于 MapReduce 的框架限制，一个 MapReduce 任务只能包含一次 Map 和一次 Reduce，计算完成之后，MapReduce 会将运算结果写回到磁盘中（更准确地说是分布式存储系统）供下次计算使用。如果所做的运算涉及大量循环，比如估计模型参数的梯度下降或随机梯度下降算法就需要多次循环使用训练数据，那么整个计算过程会不断重复地往磁盘里读写中间结果。这样的读写数据会引起大量的网络传输以及磁盘读写，极其耗时，而且它们都是没什么实际价值的废操作。因为上一次循环的结果会立马被下一次使用，完全没必要将其写入磁盘。整个算法的瓶颈是不必要的数据读写，而Spark 主要改进的就是这一点。具体地，Spark 延续了MapReduce 的设计思路：对数据的计算也分为Map 和Reduce 两类。但不同的是，一个Spark 任务并不止包含一个Map 和一个Reduce，而是由一系列的Map、Reduce构成。这样，计算的中间结果可以高效地转给下一个计算步骤，提高算法性能。虽然Spark 的改进看似很小，但实验结果显示，它的算法性能相比MapReduce 提高了10～100 倍。

另：在MapReduce 框架下，数据的格式都是key-value 形式，其中key 有两个作用：一方面它被用作统计的维度，类似于SQL 语句里面的group by 字段，比如正文例子里的字符；另一方面它又被用作数据的“指南针”决定数据将被发送到哪台机器上。而后者是分布式计算框架的核心。在某些计算场景下，计算本身不需要key 值，或者说不需要Map 这一步，比如对一个数字数组求和。这种情况下，系统会自动地生成一个Map 用于将数据转换为key-value 形式，这时key 值就只被用作数据的“指南针”
原文链接：https://www.zhihu.com/question/53354580/answer/307863620



#### Spark

* spark是处理大规模数据的通用计算引擎

* 将数据统一抽象成RDD

* Spark通过DAG（有向无环图）对RDD的关系建模，描述RDD间的依赖关系，在计算阶段实现数据共享

* Spark支持数据计算流和数据缓存

#### 

