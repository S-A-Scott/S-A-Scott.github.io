---
layout:     post
title:      "MapReduce（二）基本程序框架"
subtitle:   "Basic Framework of MR"
date:       2017-01-23 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - MapReduce
    - Hadoop
---

> 说明：本文介绍MapReduce架构和运行流程都建立在yarn上，即mapreduce.framework.name属性为yarn。

### 一. MR程序运行流程
![mapreduceOnYarn.png](/img/in-post/post-js-version/mapreduceOnYarn.png)
1. 客户端向ResourceManager申请提交application
2. ResourceManager返回资源提交路径以及applicationID
3. 客户端将作业资源复制到HDFS上
4. 客户端通过RPC调用ResourceManager的submitApplication()方法提交作业
5. ResourceManager收到调用它的submitApplication()task后，将task传递给调度器（scheduler），调度器根据相应的调度算法（例如FIFO，Fair，Capacity）执行task任务
6. 执行task时，调度器分配一个container，然后ResourceManager通过NodeManager在container中启动AppMaster
7. 从HDFS中获取作业资源
8. MRAppMaster为作业中的所有map任务和reduce任务向ResourceManager请求容器
9. scheduler分配MRAPPMaster请求的容器
10. 容器从HDFS中获取作业资源
11. MRAPPMaster通过NodeManager通信来启动容器，创建对应的Map/Reduce Task

### 一. WordCount
* Driver
```java
public class WordCountDriver {

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    // 如果打包在集群上跑 不需要设置
    // conf.set("mapreduce.framework.name", "yarn");
    // conf.set("yarn.resourcemanager.zk-address", "hadoop2:2181,hadoop3:2181,hadoop4:2181");
    Job job = Job.getInstance(conf);

    // 指定本业务job要使用的mapper/reducer业务类
    job.setMapperClass(WordCountMapper.class);
    job.setReducerClass(WordCountReducer.class);

    // 指定mapper输出数据的kv类型
    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(IntWritable.class);

    // 指定最终输出的数据的kv类型
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    // 指定job的输入原始文件所在位置
    Path inputPath = new Path("/wordcount/input");
    FileInputFormat.addInputPath(job, inputPath);
    // 指定job的输出文件所在目录
    Path outputPath = new Path("/wordcount/output");
    FileOutputFormat.setOutputPath(job, outputPath);

    // 指定本程序的jar包所在的本地路径
    // job.setJar("/home/hadoop/wc.jar"); 使用这种方式的话 不够灵活
    job.setJarByClass(WordCountDriver.class);

    // 将job中配置的相关参数，以及job所在的java类所在的jar包，提交给yarn去运行
    // job.submit();
    boolean res = job.waitForCompletion(true);
    System.exit(res ? 0 : 1);
  }
}
```

* Mapper
```java
public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

  private Text word = new Text();
  private IntWritable ONE = new IntWritable(1);

  @Override
  protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

    StringTokenizer tokenizer = new StringTokenizer(value.toString());

    while (tokenizer.hasMoreTokens()) {
        word.set(tokenizer.nextToken());
        context.write(word, ONE);
    }
  }
}
```

* Reducer
```java
public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

  @Override
  protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
    int count = 0;
    for (IntWritable value : values) {
        count += value.get();
    }
    context.write(key, new IntWritable(count));
  }
}
```

* 打包方式
![buildJar.png](/img/in-post/post-js-version/buildJar.png)
![buildArtifacts.png](/img/in-post/post-js-version/buildArtifacts.png)

* 在集群上运行
```shell
$ hadoop jar wc.jar cn.itcast.bigdata.wordcount.WordCountDriver args
```
