---
layout:     post
title:      "Zookeeper（一）zk常见应用场景"
subtitle:   "Introduce to Zookeeper"
date:       2017-04-12 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - Hadoop, Zookeeper
---

### 一. 什么是Zookeeper
* Zookeeper是为别的分布式程序服务的
* Zookeeper本身就是一个分布式程序（只要有半数以上节点存活，zk就能正常服务）
* Zookeeper提供的服务涵盖：主从协调，服务器动态上下线，统一配置管理，分布式共享锁，统一名称服务等
* 虽然说可以提供各种服务，但是Zookeeper在底层其实只提供了两个功能：
a. 管理（存储，读取）用户程序提交的数据
b. 为用户程序提供数据节点监听服务

### 二. Zookeeper常见应用场景
* 服务器状态的动态感知
![zkfunction01.png](https://github.com/S-A-Scott/S-A-Scott.github.io/blob/master/img/bigdata/zookeeper/zkfunction01.png)
考虑如下场景：
多台数据采集服务器实时，流式的从数据生产服务器上接收数据。当其中一台数据采集服务器宕机时，其它数据采集服务器如何得知这个消息，并且临时接管这台服务器的工作。
借助Zookeeper便能轻松解决这个问题：
1. 各节点向Zookeeper下的父节点（servers）注册（server01，server02，server03）
2. Zookeeper监听父节点servers，当父节点下的某个子节点（例如server02）断开连接后，Zookeeper删除server02节点信息，并向父节点下的其他子节点发送事件--父节点下的子节点发生改变。
3. 其他子节点收到信息后，可以查看是哪台节点宕机，并从存活的节点中选出一台临时接管server02的工作。

* 服务器主从选举
![zkfunction02.png](https://github.com/S-A-Scott/S-A-Scott.github.io/blob/master/img/bigdata/zookeeper/zkfunction02.png)
场景如图所示：
2台服务器组成的HA（High Available）架构，同一时刻只需要一台服务器工作。如何使得同一时刻只有一台服务器对外提供服务，并让客户端知道应该请求哪台服务器。
在分布式领域，这样的问题也是借助Zookeeper解决的
1. 2台服务器启动后向Zookeeper注册信息，并在Zookeeper上记录state（Slave，Master），ip等信息。
2. 按某个逻辑（例如注册的先后顺序），选举某台服务器当做Master，并修改state信息。
3. 客户端此时只需访问Zookeeper便知道哪台服务器是Master。
4. 当Master节点挂了后，Zookeeper会自动选择另一台节点作为Master。
