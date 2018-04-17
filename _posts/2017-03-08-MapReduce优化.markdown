---
layout:     post
title:      "MapReduce（三）优化"
subtitle:   "Optimization"
date:       2017-01-26 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - MapReduce
    - Hadoop
---

### 一. 如何处理大量小文件
* 默认情况下，TextInputFormat对任务的切片机制是按文件规划切片，不管文件多小都会是一个单独的切片，都会创建一个maptask执行。这样，如果有大量小文件，就会产生大量maptask，降低处理效率
* 处理方法
1. 在数据系统的最前端（预处理/采集），就将小文件先合并成大文件，再上传到HDFS
2. 如果已经是大量小文件在HDFS中了，可以使用另一种InputFormat来做切片（CombinerFileInputFormat），它的切片逻辑和FileInputFormat不同：它可以将多个小文件从逻辑上规划到一个切片上，这样多个小文件就可以交给一个maptask
```java
job.setInputFormatClass(CombineTextInputFormat.class);
CombineTextInputFormat.setMaxInputSplitSize(job, xxx);
CombineTextInputFormat.setMinInputSplitSize(job, xxx);
```

### 二. 压缩编码使用原则
* 运算密集型的job，少用压缩
* IO密集型的job，多用压缩


### 三. 如何处理数据倾斜
#### shuffle阶段数据分布不均匀
* 重新设计Paritioner
* 使用Combine
* mapside-join
* 增加reduce数量

### 四. MR参数优化

* 资源相关参数
    * mapreduce.map.memory.mb：一个MapTask可使用的资源上限（单位为MB），默认为1024.如果MapTask实际使用的资源量超过该值，则会被强制杀死
    * mapreduce.reduce.memory.mb：一个ReduceTask可使用的资源上限，默认为1024，如果ReduceTask实际使用的资源量超过该值，则会被强制杀死
    * mapreduce.map.java.opts：MapTask的JVM参数，可以在此配置默认的java heap size等参数，e.g "-Xmx1024m   -verbose:gc  -Xloggc:/tmp/@taskid@.gc" (@taskid@会被Hadoop框架自动换位相应的taskid），默认为“”
    * mapreduce.reduce.java.opts: ReduceTAsk的JVM参数
    * mapreduce.map.cpu.vcores：每个MapTask可使用的最多cpu core数目，默认值：1
    * mapreduce.map.cpu.vcores：每个ReduceTask可使用的最多cpu core数，默认值：1
    * mapreduce.task.io.mb：kvbuffer缓冲区大小，默认为100mb
    * mapreduce.map.sort.spill.percent：环形缓冲区溢出阈值，默认为80%

* 容错相关参数
    * mapreduce.map.maxattempts：每个MapTask最大重试次数，一旦重试参数超过该值，则认为MapTask运算失败，默认为4
    * mapreduce.reduce.maxattempts：每个ReduceTask最大重试次数，一旦重试参数超过该值，则认为ReduceTask运行失败，默认为4
    * mapreduce.map.failures.maxpercent：当失败的MapTask失败比例超过该值，整个作业则失效，默认为0。如果你的应用程序允许丢弃部分输入数据，则该值设为一个大于0的值，比如5，表示如果有低于5%的MapTask失败（如果一个MapTask重试次数超过mapreduce.map.maxattempts，则认为这个MapTask失败），整个作业任认为成功
    * mapreduce.reduce.failures.maxpercent：当失败的ReduceTask失败比例超过该值，整个作业则失效，默认为0
    * mapreduce.task.timeout：Task超时时间，经常需要设置的一个参数，该参数表达的意思为：如果一个task在一定时间内没有任何进入，即不会读取新的数据，也没有输出数据，则认为该task处于block状态，可能是卡住了，也许永远会卡主，为了防止因为用户程序永远block住不退出，则强制设置了一个超时时间（单位毫秒），默认是300000。如果你的程序对每条输入数据的处理时间过长（比如会访问数据库，通过网络拉取数据等），建议将该参数调大，该参数过小常出现的错误提示是“AttemptID:attemp_14267829456721_12456_m_000224_0 Timed out after 300 secsContainer killed by the ApplicationMaster."

* 效率和稳定性相关参数
    * mapreduce.map.speculative：是否为MapTask打开推测执行机制，默认为false
    * mapreduce.reduce.speculative：是否为ReduceTask打开推测执行机制，默认为false
    * mapreduce.job.user.classpath.first & mapreduce.task.classpath.user.precedence：当同一个class出现在用户jar包和hadoop jar中时，优先使用哪个jar包中的class，默认为false，表示优先使用hadoop jar中的class
    * mapreduce.input.fileinputformat.split.minsize: FileFormat作切片时最小切片大小
    * mapreduce.input.fileinputformat.split.maxsize：FileInputFormat作切片时的最大切片大小

* YARN配置（需要在yarn启动之前就配置在服务器的配置文件中）
    * yarn.scheduler.minimum-allocation-mb：给应用程序container分配的最小内存
    * yarn.scheduler.maximum-allocation-mb：给应用程序container分配的最大内存
    * yarn.scheduler.minimum-allocation-vcores
    * yarn.scheduler.maximum-acllocation-vcores
    * yarn.nodemanager.resource.memory-mb


