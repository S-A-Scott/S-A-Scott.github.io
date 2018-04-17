---
layout:     post
title:      "Spark SQL（三）UDF&UDAF&开窗函数"
subtitle:   "Spark SQL UDF&DFA&Window Function"
date:       2017-06-11 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spark
---

### UDF
实现一个统计名字(name)长度的UDF函数
```scala
object MyUDF {
  def main(args: Array[String]) = {
    val conf = new SparkConf().setMaster("local[*]").setAppName("MyUDF")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)

    val rdd = sc.parallelize(Array("zhangsan", "lisi", "wangwu"))
    // 转换为row RDD
    val rowRDD = rdd.map(elem => {
      Row(elem)
    })
    // 设置schema
    val schema = StructType(Array(
      StructField("name", StringType, true)
    ))
    // 创建DataFrame然后注册成表
    val df = sqlContext.createDataFrame(rowRDD, schema)
    df.registerTempTable("t_user")
    // 注册UDF函数
    sqlContext.udf.register("StrLen",
      (name: String) => name.length)
    // 使用UDF
    sqlContext.sql("select name, StrLen(name) as length from t_user").show

    sc.stop()
  }
}
```

### UDAF
自定义UDAF实现COUNT的功能，UDAF函数的参数必须出现在GROUP BY后面
```scala
// 类似Spark CombineByKey的逻辑
class MyCount extends UserDefinedAggregateFunction {
  // 该方法指定具体输入数据的类型
  override def inputSchema: StructType = {
    StructType(Array(
      StructField("input", StringType, true)
    ))
  }

  //在进行聚合操作的时候所要处理的数据的结果的类型
  override def bufferSchema: StructType = {
    StructType(Array(
      StructField("count", IntegerType, true)
    ))
  }

  //指定UDAF函数计算后返回的结果类型
  override def dataType: DataType = IntegerType

  // 确保一致性 一般用true
  override def deterministic: Boolean = true

  // 每个分组的数据执行初始化
  override def initialize(buffer: MutableAggregationBuffer): Unit = {
    buffer(0) = 0
  }

  // 在进行聚合的时候，每当有新的值进来，对分组后的聚合如何进行计算
  override def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
    buffer(0) = buffer.getAs[Int](0) + 1
  }

  // 最后merger的时候，在各个节点上的聚合值，要进行merge，也就是合并
  override def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
    buffer1(0) = buffer1.getAs[Int](0) + buffer2.getAs[Int](0)
  }

  //返回UDAF最后的计算结果
  override def evaluate(buffer: Row): Any = {
    buffer.getAs[Int](0)
  }
}
object MyUDAF {
  def main(args: Array[String]) = {
    val conf = new SparkConf().setMaster("local[*]").setAppName("MyUDAF")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)

    val rdd = sc.parallelize(Array("zhangsan", "lisi", "wangwu", "lisi", "wangwu"))
    // 转换为row RDD
    val rowRDD = rdd.map(elem => {
      Row(elem)
    })
    // 设置schema
    val schema = StructType(Array(
      StructField("name", StringType, true)
    ))
    // 创建DataFrame然后注册成表
    val df = sqlContext.createDataFrame(rowRDD, schema)
    df.registerTempTable("t_user")
    // 注册自定义的UDAF
    sqlContext.udf.register("MyCount", new MyCount())
    // 使用
    sqlContext.sql("select name, MyCount(name) as times from t_user group by name").show
    sc.stop()
  }
}
```

### 开窗函数
`row_number()`开窗函数
主要是按照某个字段分组，然后取另一个字段的前几个的值，相当于分组取topN
`row_number() over (partition by xxx order by xxx desc) xxx`
如果SQL语句里使用了开窗函数，那么这个SQL语句必须用HiveContext来执行，HiveContext默认情况下在本地无法创建
```scala

object RowNumberWindowFunction {
  def main(args: Array[String]) = {
    val conf = new SparkConf().setAppName("RowNumberWindowFunction")
    val sc = new SparkContext(conf)
    val hiveContext = new HiveContext(sc)

    hiveContext.sql("use shizhan03")
    hiveContext.sql("create table if not exists sales" +
      "(date string, category string, price Int) " +
      "row format delimited fields terminated by '\t'")
    hiveContext.sql("load data local inpath '/tmp/sales' into table sales")
    /**
      * **
      * 以类别分组，按每种类别金额降序排序，显示 【日期，种类，金额】 结果，如：
      *
      * 1 A 100
      * 2 B 200
      * 3 A 300
      * 4 B 400
      * 5 A 500
      * 6 B 600
      * 排序后：
      * 5 A 500  --rank 1
      * 3 A 300  --rank 2
      * 1 A 100  --rank 3
      * 6 B 600  --rank 1
      * 4 B 400	--rank 2
      * 2 B 200  --rank 3
      *
      */
    val result = hiveContext.sql("select date, category, price from (" +
      "select date, category, price, " +
      "row_number() over (partition by category order by price desc) as rank " +
      "from sales) as t " +
      "where t.rank <= 3")
    result.show
    sc.stop()
  }
}
```
