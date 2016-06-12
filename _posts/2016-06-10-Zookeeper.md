---
layout:     post
title:      "Zookeeper简介"
subtitle:   "分布式应用中广泛使用Zookeeper，使用Paxos为基础提供分布式一致性"
date:       2016-06-10 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Hadoop
    - Zookeeper
---

## 1、Zookeeper基本框架

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的**Chubby**一个开源的实现，是Hadoop和Hbase的重要组件。Zookeeper 作为一个分布式的服务框架，主要用来解决分布式集群中应用系统的一致性问题，它能提供基于类似于文件系统的目录节点树方式的数据存储， Zookeeper 作用主要是用来维护和监控存储的数据的状态变化，通过监控这些数据状态的变化，从而达到基于数据的集群管理。

Zookeeper集群主要角色有Leader，Learner（Follower，Observer(当服务器增加到一定程度，由于投票的压力增大从而使得吞吐量降低，所以增加了Observer。）以及client：

- Leader：领导者，负责投票的发起和决议，以及更新系统状态


- Follower：接受客户端的请求并返回结果给客户端，并参与投票


- Observer：接受客户端的请求，将写的请求转发给leader，不参与投票。Observer目的是扩展系统，提高读的速度。


- Client:客户端，向Zookeeper发起请求。

Zookeeper的基本框架图如下：

![](http://img.blog.csdn.net/20150310184459281?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVucWlu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### Leader的主要功能：

- 恢复数据；


- 维持与Learner的心跳，接收Learner请求并判断Learner的请求消息类型；Learner的消息类型主要有PING消息、REQUEST消息、ACK消息、REVALIDATE消息，根据不同的消息类型，进行不同的处理。PING消息是指Learner的心跳信息；REQUEST消息是Follower发送的提议信息，包括写请求及同步请求；ACK消息是Follower的对提议的回复，超过半数的Follower通过，则commit该提议；REVALIDATE消息是用来延长SESSION有效时间。

![](http://img.blog.csdn.net/20150310184541151?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVucWlu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### Follower基本功能：

向Leader发送请求（PING消息、REQUEST消息、ACK消息、REVALIDATE消息）；

接收Leader消息并进行处理；

接收Client的请求，如果为写请求，发送给Leader进行投票；

![](http://img.blog.csdn.net/20150310184614052?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVucWlu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**Observer主要功能同Follower的唯一不同的地方就是observer不会参加leader发起的投票。**

#### Zookeeper配置介绍

![](http://img.blog.csdn.net/20150310184703270?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVucWlu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- tickTime ：基本事件单元，以毫秒为单位。这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。


- dataDir ：存储内存中[数据库](http://lib.csdn.net/base/14)快照的位置，顾名思义就是 Zookeeper 保存数据的目录，默认情况下，Zookeeper 将写数据的日志文件也保存在这个目录里。


- clientPort ：这个端口就是客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。


- initLimit：这个配置项是用来配置 Zookeeper 接受客户端初始化连接时最长能忍受多少个心跳时间间隔数，当已经超过 5 个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒。


- syncLimit：这个配置项标识 Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 2*2000=4 秒


- server.A = B:C:D : A表示这个是第几号服务器,B 是这个服务器的 ip 地址；C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader。

#### Zookeeper集群下的基本操作

查看各Zookeeper服务角色：

![](http://img.blog.csdn.net/20150310184736279?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVucWlu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

基本命令操作：

![](http://img.blog.csdn.net/20150310184925478?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVucWlu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 2、Zookeeper数据结构

Zookeeper 的数据结构是一个树形结构 ，非常类似于一个标准的文件系统。每个子节点项都有唯一的路径标识，如 Server1 节点的标识为 /NameService/Server1。

![](http://img.blog.csdn.net/20150310184846667?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXVucWlu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### Znode

Zookeeper数据结构中每个节点称为Znode，每个Znode都有唯一的路径，znode 可以有子节点目录，并且每个 znode 可以存储数据。znode是有版本的，每个 znode 中存储的数据可以有多个版本，也就是一个访问路径中可以存储多份数据。

Znode 基本类型 ：

- PERSISTENT：持久化znode节点，一旦创建这个znode点存储的数据不会主动消失，除非是客户端主动的delete。


- PERSISTENT|SEQUENTIAL：顺序自动编号的znode节点，这种znoe节点会根据当前已近存在的znode节点编号自动加 1，而且不会随session断开而消失。


- EPHEMERAL：临时znode节点，Client连接到zk service的时候会建立一个session，之后用这个zk连接实例创建该类型的znode，一旦Client关闭了zk的连接，服务器就会清除session，然后这个session建立的znode节点都会从命名空间消失。总结就是，这个类型的znode的生命周期是和Client建立的连接一样的。


- PHEMERAL|SEQUENTIAL：临时自动编号节点，znode节点编号会自动增加，但是会随session消失而消失。

Zookeeper它只负责协调数据，一般 Znode上的数据都比较小以Kb为测量单位。Zookeeper的client和server的实现类都会验证znode存储的数据是否小于1M。如果数据比较大时，Server之间进行数据同步会消耗比较长的时间，影响系统性能。

#### Watcher

Zookeeper中znode产生某种行为（事件）时，如何让客户端得到通知，进行相关操作？Zookeeper中使用Watcher机制，可以针对ZooKeeper服务的“操作”来设置观察，该服务的其他操作可以触发观察。

Zookeeper中的watcher机制类型：

- Exists:在path上执行NodeCreated ,NodeDeleted ,NodeDataChanged .


- getData Watcher: 在path上执行 NodeDataChanged ,NodeDeleted .


- getChildrenWatcher:在paht上执行NodeDeleted .或在子path上执行NodeCreated ,NodeDeleted 。

Zookeeper中对于某个节点设置Watcher是一次性的，在Znode上watcher触发后会删除该Watcher，所以如果需要对某个Znode节点进行长期关注，在事件触发后，需要在该Znode上重置Watcher。

#### Zookeeper基本操作示例

```java
 // 创建一个与服务器的连接
 ZooKeeper zk = new ZooKeeper("localhost:" + CLIENT_PORT, 
        ClientBase.CONNECTION_TIMEOUT, new Watcher() { 
            // 监控所有被触发的事件
            public void process(WatchedEvent event) { 
                System.out.println("已经触发了" + event.getType() + "事件！"); 
            } 
        }); 
 // 创建一个目录节点
 zk.create("/testRootPath", "testRootData".getBytes(), Ids.OPEN_ACL_UNSAFE,
   CreateMode.PERSISTENT); 
 // 创建一个子目录节点
 zk.create("/testRootPath/testChildPathOne", "testChildDataOne".getBytes(),
   Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT); 
 System.out.println(new String(zk.getData("/testRootPath",false,null))); 
 // 取出子目录节点列表
 System.out.println(zk.getChildren("/testRootPath",true)); 
 // 修改子目录节点数据
 zk.setData("/testRootPath/testChildPathOne","modifyChildDataOne".getBytes(),-1); 
 System.out.println("目录节点状态：["+zk.exists("/testRootPath",true)+"]"); 
 // 创建另外一个子目录节点
 zk.create("/testRootPath/testChildPathTwo", "testChildDataTwo".getBytes(), 
   Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT); 
 System.out.println(new String(zk.getData("/testRootPath/testChildPathTwo",true,null))); 
 // 删除子目录节点
 zk.delete("/testRootPath/testChildPathTwo",-1); 
 zk.delete("/testRootPath/testChildPathOne",-1); 
 // 删除父目录节点
 zk.delete("/testRootPath",-1); 
 // 关闭连接
 zk.close();
```



## 3、Zookeeper的基本应用

Zookeeper 从设计模式角度来看，是一个基于**观察者模式**设计的分布式服务管理框架，它负责存储和管理大家都关心的数据，然后接受观察者的注册，一旦这些数据的状态发生变化，Zookeeper 就将负责通知已经在 Zookeeper 上注册的那些观察者做出相应的反应，从而实现集群中类似 Master/Slave 管理模式，关于 Zookeeper 的详细架构等内部细节可以阅读 Zookeeper 的源码。

下面详细介绍这些典型的应用场景

#### 统一命名服务

分布式应用中，通常需要有一套完整的命名规则，既能够产生唯一的名称又便于人识别和记住，通常情况下用树形的名称结构是一个理想的选择，树形的名称结构是一个有层次的目录结构，既对人友好又不会重复。说到这里你可能想到了 JNDI，没错 Zookeeper 的 Name Service 与 JNDI 能够完成的功能是差不多的，它们都是将有层次的目录结构关联到一定资源上，但是 Zookeeper 的 Name Service 更加是广泛意义上的关联，也许你并不需要将名称关联到特定资源上，你可能只需要一个不会重复名称，就像数据库中产生一个唯一的数字主键一样。

Name Service 已经是 Zookeeper 内置的功能，你只要调用 Zookeeper 的 API 就能实现。如调用 create 接口就可以很容易创建一个目录节点。

#### 配置管理

配置的管理在分布式应用环境中很常见，例如同一个应用系统需要多台 PC Server 运行，但是它们运行的应用系统的某些配置项是相同的，如果要修改这些相同的配置项，那么就必须同时修改每台运行这个应用系统的 PC Server，这样非常麻烦而且容易出错。

像这样的配置信息完全可以交给 Zookeeper 来管理，将配置信息保存在 Zookeeper 的某个目录节点中，然后将所有需要修改的应用机器监控配置信息的状态，一旦配置信息发生变化，每台应用机器就会收到 Zookeeper 的通知，然后从 Zookeeper 获取新的配置信息应用到系统中。

![](http://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/image002.gif)

#### 集群管理

Zookeeper 能够很容易的实现集群管理的功能，如有多台 Server 组成一个服务集群，那么必须要一个“总管”知道当前集群中每台机器的服务状态，一旦有机器不能提供服务，集群中其它集群必须知道，从而做出调整重新分配服务策略。同样当增加集群的服务能力时，就会增加一台或多台 Server，同样也必须让“总管”知道。

Zookeeper 不仅能够帮你维护当前的集群中机器的服务状态，而且能够帮你选出一个“总管”，让这个总管来管理集群，这就是 Zookeeper 的另一个功能 Leader Election。

它们的实现方式都是在 Zookeeper 上创建一个 EPHEMERAL 类型的目录节点，然后每个 Server 在它们创建目录节点的父目录节点上调用[getChildren](http://hadoop.apache.org/zookeeper/docs/r3.2.2/api/org/apache/zookeeper/ZooKeeper.html#getChildren%28java.lang.String,%20boolean%29)([String](http://java.sun.com/javase/6/docs/api/java/lang/String.html?is-external=true) path, boolean watch) 方法并设置 watch 为 true，由于是 EPHEMERAL 目录节点，当创建它的 Server 死去，这个目录节点也随之被删除，所以 Children 将会变化，这时 [getChildren](http://hadoop.apache.org/zookeeper/docs/r3.2.2/api/org/apache/zookeeper/ZooKeeper.html#getChildren%28java.lang.String,%20boolean%29)上的 Watch 将会被调用，所以其它 Server 就知道已经有某台 Server 死去了。新增 Server 也是同样的原理。

Zookeeper 如何实现 Leader Election，也就是选出一个 Master Server。和前面的一样每台 Server 创建一个 EPHEMERAL 目录节点，不同的是它还是一个 SEQUENTIAL 目录节点，所以它是个 EPHEMERAL_SEQUENTIAL 目录节点。之所以它是 EPHEMERAL_SEQUENTIAL 目录节点，是因为我们可以给每台 Server 编号，我们可以选择当前是最小编号的 Server 为 Master，假如这个最小编号的 Server 死去，由于是 EPHEMERAL 节点，死去的 Server 对应的节点也被删除，所以当前的节点列表中又出现一个最小编号的节点，我们就选择这个节点为当前 Master。这样就实现了动态选择 Master，避免了传统意义上单 Master 容易出现单点故障的问题。

![](http://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/image003.gif)

#### 共享锁（Locks）

共享锁在同一个进程中很容易实现，但是在跨进程或者在不同 Server 之间就不好实现了。Zookeeper 却很容易实现这个功能，实现方式也是需要获得锁的 Server 创建一个 EPHEMERAL_SEQUENTIAL 目录节点，然后调用 [getChildren](http://hadoop.apache.org/zookeeper/docs/r3.2.2/api/org/apache/zookeeper/ZooKeeper.html#getChildren%28java.lang.String,%20boolean%29)方法获取当前的目录节点列表中最小的目录节点是不是就是自己创建的目录节点，如果正是自己创建的，那么它就获得了这个锁，如果不是那么它就调用 [exists](http://hadoop.apache.org/zookeeper/docs/r3.2.2/api/org/apache/zookeeper/ZooKeeper.html#exists%28java.lang.String,%20boolean%29)([String](http://java.sun.com/javase/6/docs/api/java/lang/String.html?is-external=true) path, boolean watch) 方法并监控 Zookeeper 上目录节点列表的变化，一直到自己创建的节点是列表中最小编号的目录节点，从而获得锁，释放锁很简单，只要删除前面它自己所创建的目录节点就行了。

![](http://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/image004.gif)

#### 队列管理

Zookeeper 可以处理两种类型的队列：

1. 当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达，这种是同步队列。
2. 队列按照 FIFO 方式进行入队和出队操作，例如实现生产者和消费者模型。

同步队列用 Zookeeper 实现的实现思路如下：

创建一个父目录 /synchronizing，每个成员都监控标志（Set Watch）位目录 /synchronizing/start 是否存在，然后每个成员都加入这个队列，加入队列的方式就是创建 /synchronizing/member_i 的临时目录节点，然后每个成员获取 / synchronizing 目录的所有目录节点，也就是 member_i。判断 i 的值是否已经是成员的个数，如果小于成员个数等待 /synchronizing/start 的出现，如果已经相等就创建 /synchronizing/start。

![](http://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/image005.gif)

**生产者和消费者这种队列形式的示例代码：**

```java
//生产者代码
boolean produce(int i) throws KeeperException, InterruptedException{ 
        ByteBuffer b = ByteBuffer.allocate(4); 
        byte[] value; 
        b.putInt(i); 
        value = b.array(); 
        zk.create(root + "/element", value, ZooDefs.Ids.OPEN_ACL_UNSAFE, 
                    CreateMode.PERSISTENT_SEQUENTIAL); 
        return true; 
    }

//消费者代码
int consume() throws KeeperException, InterruptedException{ 
        int retvalue = -1; 
        Stat stat = null; 
        while (true) { 
            synchronized (mutex) { 
                List<String> list = zk.getChildren(root, true); 
                if (list.size() == 0) { 
                    mutex.wait(); 
                } else { 
                    Integer min = new Integer(list.get(0).substring(7)); 
                    for(String s : list){ 
                        Integer tempValue = new Integer(s.substring(7)); 
                        if(tempValue < min) min = tempValue; 
                    } 
                    byte[] b = zk.getData(root + "/element" + min,false, stat); 
                    zk.delete(root + "/element" + min, 0); 
                    ByteBuffer buffer = ByteBuffer.wrap(b); 
                    retvalue = buffer.getInt(); 
                    return retvalue; 
                } 
            } 
        } 
 }
```



参考：[zookeeper研究笔记（一）—— single模式搭建](http://blog.csdn.net/lastsweetop/article/details/8791961)

[zookeeper研究笔记（二）—— 集群模式搭建](http://blog.csdn.net/lastsweetop/article/details/8794473)

[ZooKeeper 基本介绍](http://blog.csdn.net/qunqin/article/details/44179035)

[分布式服务框架 Zookeeper](http://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/#icomments)






























