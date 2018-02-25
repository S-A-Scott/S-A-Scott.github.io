---
layout:     post
title:      "MapReduce（一）运行流程"
subtitle:   "Progress of MR"
date:       2017-01-21 12:00:00
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
