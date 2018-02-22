---
layout:     post
title:      "Zookeeper（二）zk特性和数据模型"
subtitle:   "Zookeeper Data Model"
date:       2017-04-13 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - Zookeeper
---

### 一. Zookeeper特性
* 是由一个leader，多个follower组成的集群
* 数据一致性，每个server保存一份相同的数据副本，client无论连到哪个server，数据都是一致的。在Zookeeper的设计中，以下几点考虑保证了数据的一致性：
  1. 顺序一致性：更新请求顺序进行，来自同一个client的更新请求按其发送顺序依次执行
  2. 原子性：一次数据更新要么成功，要么失败
  3. 实时性：在一定时间范围内，client能读到最新数据
  4. 持久性：一个更新一旦成功，其结果就会持久存在并且不会被撤销
  5. 单一系统映像：一个客户端无论连接到哪一台服务器，它看到的都是同样的系统视图
  
### 二. Zookeeper数据模型
* 层次化的树形结构
* 每个节点在Zookeeper中称为znode，并具有唯一路径标识
* znode既可以保存数据，也可以保存其他znode（子节点）
* znode以某种方式发生变化时，观察（watch）机制可以让客户端得到通知
![zkstruct.png](/img/in-post/post-js-version/zkstruct.png)

### 三. 节点类型
* Znode有两种类型：
  1. 短暂（ephemeral）创建znode的客户端断开连接时，短暂znode都会被Zookeeper服务删除。并且短暂znode不可以有子节点
  2. 持久（persistent）创建znode的客户端断开连接时，持久znode不会被删除
* Znode有四种形式的目录节点（默认为persistent）
  1. persistent
  2. persistent_sequential
  3. ephemeral
  4. ephemeral_sequential
* 创建znode时设置顺序号（sequential），znode名称后会附加一个值，这个值是由一个单迪增的计数器（由父节点维护）所添加的
