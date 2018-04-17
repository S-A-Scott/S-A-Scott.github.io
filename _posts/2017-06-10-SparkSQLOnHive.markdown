---
layout:     post
title:      "Spark SQL（二）Spark On Hive"
subtitle:   "Spark SQL On Hive"
date:       2017-06-10 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spark
---

> hadoop1是hive存放元数据mysql服务器
> hadoop3是启动metastore的hive服务器
> hadoop4是hive client

### 概念

* Spark on Hive
  * Hive只是作为存储的角色
  * SparkSQL作为计算的角色

* Hive on Spark
  * Hive承担了一部分计算（解析SQL，优化SQL...）的和存储
  * Spark作为了执行引擎的角色

### 配置
* 将hadoop4上的hive-site.xml放入hadoop4上的`$SPARK_HOME/conf`目录下
* 将hadoop3上`$HIVE_HOME/lib/mysql-connector.jar`拷贝到hadoop4下的相同目录
* hadoop3启动`hive --service metastore`
* hadoop4启动spark-shell
```shell
./spark-shell --master spark://hadoop1:7077 --driver-class-path /usr/local/apache-hive-1.2.1-bin/lib/mysql-connector-java-5.1.32-bin.jar
```

* 使用sqlContext调用HQL
```scala
sqlContext.sql("select count(*) from tbl1").show
```

* 或者使用hiveContext
```scala
import org.apache.spark.sql.hive.HiveContext
val hiveContext = new HiveContext(sc)
hiveContext.sql("select * from tbl1").show
```


### 注意

如果使用Spark on Hive查询数据时，出现错误
```shell
Caused by: java.net.UnknownHostException: XXX
```
找不过HDFS集群路径，需要在客户端机器`conf/spark-env.sh`中设置HDFS路径：
```shell
export HADOOP_CONF_DIR=$HADOOP_PREFIX/etc/hadoop
```
