---
layout:     post
title:      "Zookeeper（三）Java API"
subtitle:   "Zookeeper API"
date:       2017-04-13 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - Zookeeper
---

### 一. 基本操作
* org.apache.zookeeper.Zookeeper是客户端入口主类，负责建立与server的会话，提供了下表中的9种操作

| 操作             | 描述                                    |
| ---------------- |---------------------------------------- |
| create           | 创建一个znode                           |
| delete           | 删除一个znode（该znode不能有任何子节点）|
| exists           | 测试一个znode是否存在并查询它的元数据   |
| getACL, setACL   | 获取/设置一个znode的ACL                 |
| getChildren      | 获取一个znode的子节点列表               |
| getData, setData | 获取/设置一个znode所保存的数据          |
| sync             | 将客户端的znode视图与Zookeeper同步      |

### 二. 常用API
* 建立连接，创建节点
```java
public class SimpleZkClient {
    private static final String connectString =
          "hadoop2:2181,hadoop3:2181,hadoop4:2181";
    private static final int sessionTimeout = 2000;
    ZooKeeper zkClient = null;
    @Before
    public void init() throws Exception {
    //构造方法Zookeeper(String connectString, int sessionTimeout, Watcher watcher)
    //Watcher接口中需要实现 收到事件通知后的回调函数
    //                 ----- abstract public void process(WatchedEvent event);
    //回调函数收到一次事件通知后就失效，如果想处理多次事件，需要在回调函数中再次注册Watcher
        zkClient = new ZooKeeper(connectString, sessionTimeout, event ->
        System.out.println(event.getType() + "------" + event.getPath()));
    }
    @Test
    public void testCreate() throws KeeperException, InterruptedException {
      //创建一个znode
      //参数1: 要创建的节点
      //参数2: 节点数据
      //参数3: 节点的权限
      //参数4: 节点的类型
        String nodeCreated = zkClient.create("/intellij",
                "It_this_node_data".getBytes(),
                Ids.OPEN_ACL_UNSAFE,
                CreateMode.PERSISTENT);
    }
}
```

* 获取子节点
```java
  @Test
  public void getChildren() throws KeeperException, InterruptedException {
    // List<String>  getChildren(String path, boolean watch)
    List<String> children = zkClient.getChildren("/", true);
    children.forEach(System.out::println);
  }
```

* 判断是否存在
```java
  @Test
  public void testExist() throws KeeperException, InterruptedException {
      // Stat exists(String path, boolean watch)
    Stat stat = zkClient.exists("/intellij", false);
    System.out.println(stat == null ? "not exist" : "exist");
  }
```

### 三. 服务器上下线动态感知
* 考虑如下场景：
集群中服务器会有动态上下线的情况，客户端怎么知道当前可用的服务器。可以使用Zookeeper完成业务需求
![zkgetserversinfo.png](/img/in-post/post-js-version/zkgetserversinfo.png)
1. 服务器启动时，即去Zookeeper注册信息（短暂序列化znode）
2. 客户端启动后通过Zookeeper获取当前在线服务器列表，并且注册监听（通过getChildren方法）

* 简单的实现如上功能
1. 服务器实现
```java
public class DistributeServer {
    private static final String connectString =
          "hadoop2:2181,hadoop3:2181,hadoop4:2181";
    private static final int sessionTimeout = 2000;
    private static final String parentNode = "/servers";
    private ZooKeeper zk = null;
    public static void main(String[] args) throws Exception {
    // 获取zk连接
    DistributeServer server = new DistributeServer();
    server.getConnect();
    // 利用zk连接注册服务器信息
    server.registerServer(args[0]);
    // 注册完Zookeeper信息后，服务器处理自己原本的业务便可
    System.out.println(args[0] + " is doing its business...");
    Thread.sleep(Long.MAX_VALUE);
  }
    public void getConnect() throws IOException {
        zk = new ZooKeeper(connectString, sessionTimeout, event -> {
          System.out.println(event.getType() + "------" + event.getPath());
          try {
            // 再次注册事件
            zk.getChildren("/", true);
          } catch (Exception e) {
            e.printStackTrace();
          }
        });
    }
    public void registerServer(String hostname) throws KeeperException, InterruptedException {
        String createdNode = zk.create(parentNode + "/server", hostname.getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        System.out.println(hostname + " is online.." + createdNode);
    }
}
```
2. 客户端实现
```java
public class DistributeClient {
    private static final String connectString =
          "hadoop2:2181,hadoop3:2181,hadoop4:2181";
    private static final int sessionTimeout = 2000;
    private static final String parentNode = "/servers";
    private List<String> serverList;
    private ZooKeeper zk = null;
    public static void main(String[] args) throws Exception {
        // 获取zk连接
        DistributeClient client = new DistributeClient();
        client.getConnect();
        // 利用zk连接注册服务器信息
        client.getServerList();
        // 获取服务器列表后客户端继续处理自己原本的业务便可
        System.out.println("client is doing its business...");
        Thread.sleep(Long.MAX_VALUE);
    }
    public void getConnect() throws IOException {
        zk = new ZooKeeper(connectString, sessionTimeout, event -> {
            System.out.println(event.getType() + "------" + event.getPath());
            try {
            // 更新服务器列表，再次注册事件
            getServerList();
            } catch (Exception e) {
            e.printStackTrace();
            }
        });
    }
    public void getServerList() throws KeeperException, InterruptedException {
        // 获取服务器子节点信息，并监听父节点
        List<String> children = zk.getChildren("/servers", true);
        // 获取节点数据（这里子节点保存的数据是对应主机的主机名），存入局部list中
        List<String> servers  = new ArrayList<>();
        for (String child : children) {
          byte[] data = zk.getData(parentNode + "/" + child, false, null);
          servers.add(new String(data));
        }
        serverList = servers;
        serverList.forEach(System.out::println);
    }
}
```

### 四. 分布式共享锁
* 在分布式系统中，如何保证同一时刻，只有一个客户端程序能够使用某一共享资源。因为这些线程运行在不同节点上，所以不能使用传统的java同步方式（如synchronized）。在分布式领域，这样的问题，通常可以通过引入第三方解决（这里使用的是Zookeeper，当然也有其他第三方能解决这个问题）
![zksharedLock.png](/img/in-post/post-js-version/zksharedLock.png)
1. 程序启动时到zk上注册一个ephemeral_sequential类型的znode，并监听父节点
2. 获取父节点下的所有子节点，比较序号的大小
3. 序号最小的获得“锁”， 访问资源后删除自己的节点（相当于释放），并重新注册一个新的子节点
4. 其他程序收到事件通知，则可以去zk上获取锁。
```java
public class DistributeClientLock {
    private static final String connectString =
          "hadoop2:2181,hadoop3:2181,hadoop4:2181";
    private static final int sessionTimeout = 2000;
    private String groupNode = "locks";
    private String subNode   = "sub";
    private ZooKeeper zk = null;
    // 记录自己创建子节点的路径
    private String thisPath = null;
    public static void main(String[] args) throws Exception {
        // 建立连接
        DistributeClientLock distributeClientLock = new DistributeClientLock();
        distributeClientLock.getConnect();
        Thread.sleep(Long.MAX_VALUE);
    }
    public void getConnect() throws IOException, KeeperException, InterruptedException {
        zk = new ZooKeeper(connectString, sessionTimeout, event -> {
            try {
                // 判断时间类型，只处理子节点变化事件
                if (event.getType() == EventType.NodeChildrenChanged
                      && event.getPath().equals("/" + groupNode)) {
                    // 获取子节点，并对父节点进行监听
                    List<String> childrenNodes = zk.getChildren("/" + groupNode, true);
                    String thisNode = thisPath.substring(("/" + groupNode + "/").length());
                    // 比较自己是否是最小id，如果是最小ID获得"锁"，访问资源
                    Collections.sort(childrenNodes);
                    if (childrenNodes.indexOf(thisNode) == 0) {
                    // 访问共享资源并且删除"锁"
                    doSomething();
                    // 重新注册一把"锁"
                    thisPath = zk.create("/" + groupNode + "/" + subNode, null,
                            Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
                  }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        thisPath = zk.create("/" + groupNode + "/" + subNode, null,
                Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        // 如果当前只有该程序访问资源，则可以直接使用资源
        List<String> childrenNodes = zk.getChildren("/" + groupNode, true);
        if (childrenNodes.size() == 1) {
            doSomething();
            thisPath = zk.create("/" + groupNode + "/" + subNode, null,
              Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        }
    }
    private void doSomething() throws KeeperException, InterruptedException {
        System.out.println("获得锁: " + thisPath);
        System.out.println("访问资源ing...");
        Thread.sleep(5000);
        zk.delete(this.thisPath, -1);
    }
}
```
