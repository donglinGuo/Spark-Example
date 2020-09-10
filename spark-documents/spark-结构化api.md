## Spark 结构化API

1. Dataset类型
2. DataFrame类型
3. SQL表和视图

#### 结构化数据

* spark中的结构化数据类似于具有行和列的分布式表格，每列的数据类型相同，每行有相同的字段，使用null指定缺省值，因此可以通过指定行和列对特定的数据进行操作，因为保持spark数据结构只读不可变和惰性执行的特点，所以指定对结构化数据定义转化操作流，待遇到action操作时再执行。

* 类型化和非类型化，结构化API中，DataFrame是非类型化的，这里非类型化并不是没有类型而是数据中定义了类型，spark仅负责在运行时检查类型是否一致;Dataset是类型化的，spark在编译时就检查类型是否一致。

* schema，既然spark中结构化数据定义了行与列及类型，那就需要以某种方式标识它们，schema类似与表头，定义数据列与类型，schema可以从数据源中读取也可以在spark中指定。

* 行，一行对应一个数据记录record

* 列，每一列对应一个简单类型或复杂类型

* 类型，为了提高优化空间spark通过Catalyst来维护自己的内部类型，不同语言的数据类型与spark内部类型都有对应关系，并且可以直接在其它语言中使用spark类型。
  * scala中使用spark类型 `import org.apache.spark.sql.types._`
  * java中使用spark类型 `import org.apache.spark.sql.types.DataTypes;`
  * scala中使用spark类型 `from pyspark.sql.types import *`

* 结构化API执行概述
  * 用户代码->未被解析的逻辑计划->解析后的逻辑计划->优化的逻辑计划
    这部分主要是对用户的代码所表示的操作进行优化
  * 优化的逻辑计划->不同的物理执行策略->代价模型比较分析->最佳物理执行计划
    这部分工作是根据优化后的逻辑计划生成最有的物理执行计划即生成一些列的RDD和转换操作
  * 执行

#### 结构化操作

* DataFrame
  DataFrame是由record组成，record是Row类型，一个record由多列组成，模式定义了列名及数据类型，partition定义了数据在集群上的物理分布，划分模式定义了partition带的分配方式。

  * schema
    schema可以由数据源来定义也可以由我们显式定义
    * schema的形式 \
      schema由许多StructField构成的StructType，每一个SturctField描述一个列的信息，包括列名称、列类型（spark内部类型）、布尔标识（是否可以包含缺失值）、与该列关联的metadata信息构成。\
        ```scala
        // 读取源文件的schema
        spark.read.format("json").load("/data/flight-data/json/2015-summary.json").schema

        // 定义schema
        import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}
        import org.apache.spark.sql.types.Metadata
        val myManualSchema = StructType(Array(
            StructField("DEST_COUNTRY_NAME", StringType, true),
            StructField("ORIGIN_COUNTRY_NAME", StringType, true),
            StructField("count", LongType, false,
            Metadata.fromJson("{\"hello\":\"world\"}"))
        ))
        val df = spark.read.format("json").schema(myManualSchema)
            .load("/data/flight-data/json/2015-summary.json")
        ```
  * 列和表达式
    * 列,在spark中列是一个逻辑结构，它只是根据表达式所定义的选择、转换、删除等操作为每个记录计算出的值，因此它是建立在DataFrame上的表达式，所以可以认为表达式即为列。
      * 列的构造和引用 \
        通过col函数或则column函数或则其它引用符号 
        ```scala
        // in Scala
        import org.apache.spark.sql.functions.{col, column}
        col("someColumnName")
        column("someColumnName")

        // scala特有的引用符号,$将字符串指定为表达式，“’”符号指定一个symbol，是scala引用标识符的特殊结构
        $"myColumn" 
        'myColumn
        ```
    * 表达式，表达式是对一个转换函数，输入是一个或多个列，输出是一个"值"，最简单情况下表达式就是列的引用`expr("colName")`，当使用表达式时，expr函数实际上可以将字符串解析成转换操作和列的引用，也可以将结果传递到下一个转换操作,以下操作是等价的。
      ```scala
        expr("someCol - 5")
        col("someCol") - 5
        expr("someCol") -5
      ```
    * 列与对这些列的操作被编译后生成的逻辑计划与解析后的表达式的逻辑计划是一样的。以下操作是等价的，需要注意的是该表达式是有效的SQL代码，因为SQL与DataFrame代码在执行前都会编译成相同的底层逻辑树，所以SQL表达式和DataFrame代码性能是一样的。
      ```scala
        (((col("someCol") + 5) * 200) - 6) < col("otherCol")

        import org.apache.spark.sql.functions.expr
        expr("(((someCol + 5) * 200) - 6) < otherCol")
      ```
  * 记录和行
      