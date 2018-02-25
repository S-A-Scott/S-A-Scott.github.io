---
layout:     post
title:      "Hadoop RPC API"
subtitle:   "Basic Use of RPC API"
date:       2017-01-21 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - RPC
    - Hadoop
---

### 一. 简述
* 本文使用Hadoop的RPC框架简略模拟了HDFS客户端向NameNode查询元数据的过程
* 项目结构如下
![hadooprpc.png](/img/in-post/post-js-version/hadooprpc.png)

### 二. 代码实现
* 通信协议
```java
public interface ClientNamenodeProtocol {
    // 定义协议版本号
    public static final long versionID = 1L;
    // 定义通信接口
    public String getMetaData(String path);
}
```

* 服务器端实现业务功能
```java
public class MyNameNode implements ClientNamenodeProtocol {
  // 模拟查询元数据
    @Override
    public String getMetaData(String path) {
        return "This is  meta data from " + path +
            " : replication - x, block locations - xxx";
    }
}
```

* 发布服务器
```java
public class PublishServiceUtil {
    public static void main(String[] args) throws IOException {
        RPC.Builder builder = new RPC.Builder(new Configuration());
        builder.setBindAddress("localhost")
                .setPort(8888)
                .setProtocol(ClientNamenodeProtocol.class)
                .setInstance(new MyNameNode());

        RPC.Server server = builder.build();
        server.start();
    }
}
```

* 客户端
```java
public class MyHDFSClient {

    public static void main(String[] args) throws IOException {
        InetSocketAddress inetSocketAddress = new InetSocketAddress("localhost", 8888);
        ClientNamenodeProtocol nameNode = RPC.getProxy(ClientNamenodeProtocol.class, 1L, inetSocketAddress, new Configuration());

        String metaData = nameNode.getMetaData("/users/root/hello.txt");
        System.out.println(metaData);
    }
}
```
