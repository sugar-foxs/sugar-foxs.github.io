---
layout:     post
title:      zookeeper学习
subtitle:   
date:       2020-03-22
author:     sugar-foxs
catalog: 	true
tags:
    - 分布式
---

本文是关于zookepper学习的记录。
<!-- more -->

## Paxos协议
需要解决的问题是：如何在一个可能发生机器宕机或者网络异常等情况的分布式系统中，快速且准确的在集群内部对某个数据的值达到一致。并且保证不论发生以上任何异常，都不会破坏整个系统的一致性。

- 拜占庭将军问题：拜占庭帝国有很多军队，每个军队之间将军必须指定一致的行动计划，各个军队在地理上是分开的，只能依靠军队的通讯员进行通讯，但是通讯员可能会有叛徒，可以任意篡改消息，从而欺骗将军。

在实际工程实践中，可以假设不存在拜占庭问题，即假设所有消息是完整的，是没有被篡改的。因为：
  - 大部分系统都是部署在同一个局域网内，因此消息被篡改的情况非常罕见。
  - 由于硬件或者网络造成的消息不完整问题，可以使用校验算法避免。

  Paxos算法描述：Paxos小岛采用议会的形式通过法令，议员通过信使来进行消息的传递，值得注意的是，议员和信使都是兼职的，他们随时有可能离开议会厅，并且信使可能会重复的传递消息，也可能一去不复返，因此，议会协议要保证 这种情况下，法令仍然能正确的产生，并且不会出现冲突。

- google chubby是Paxos的实现。

## ZAB协议
- zookeeper是google chubby的开源实现，但是并没有直接用Paxos算法，而采用了ZAB的一致性协议。
- ZAB协议是为分布式协调服务Zookeeper专门设计的一种支持崩溃恢复的原子广播协议。

- ZAB协议的核心：所有事务请求必须由一个全局唯一的服务器爱协调处理，这样的服务器被称为Leader服务器，而余下的其他服务器则成为Follower服务器。Leader服务器负责将一个客户端事务请求转换成一个事务Proosal,并将该Proposal分发给集群中其他的Follower服务器。之后Leader服务器需要等待所有Follower服务器的反馈，一旦超过半数的Follower服务器进行了正确的反馈后，那么Leader就会再次向所有的Follower服务器分发Commit消息，要求其将前一个Proposal进行提交。

- ZAB协议包括两种基本的模式：崩溃恢复和消息广播。
- 消息广播
针对客户端的事务请求，Leader服务器会为其生成Proposal，并将其发送给集群中其他机器，然后再分别收集各自的选票，最后进行事务提交。ZAB协议在集群中过半Follower机器已经反馈ACK之后，就开始提交事务了，而不需要等待所有Follower都反馈。所以需要崩溃恢复来解决Leader服务器崩溃退出而导致的数据不一致的问题。在广播事务Proposal之前，Leader服务器会先为这个事务生成一个全局递增的诶一ID，称为事务ID。Leader服务器会为每个Follower分配一个队列，需要广播的事务放入对应的队列就可以发送出去。

- 崩溃恢复
消息广播在正常情况下运行非常良好，但是当Leader服务器出现崩溃，或者由于网络原因Leader服务器失去了与过半Follower的联系，便会进入崩溃恢复模式。即重新选举出一个Leader。 
  - 需要确保已经在Leader服务器上提交的事务，最终被所有服务器都提交。
  - 需要确保丢弃那些只在Leader服务器上被提出的事务。
- Leader选举算法：让具有最高事务编号的机器成为Leader。可以保证新选举出来的Leader服务器一定具有所有已经提交的提案。
- 数据同步：Leader选举之后，

## ZooKeeper
- ZooKeeper致力于提供一个高性能、高可用，且具有严格的顺序访问控制能力的分布式协调服务。

### 服务器角色
- ZooKeeper 集群中的所有机器通过一个 Leader 选举过程来选定一台称为 “Leader” 的机器，Leader既可以为客户端提供写服务又能提供读服务。除了Leader外，Follower 和 Observer 都只能提供读服务。
Follower 和 Observer 唯一的区别在于 Observer 机器不参与 Leader 的选举过程，也不参与写操作的“过半写成功”策略，因此 Observer 机器可以在不影响写性能的情况下提升集群的读性能。

#### Leader
- 使用责任链模式来处理每一个请求，请求处理链包含以下几个：
  - PreRequestProcessor: 请求预处理器。对于一些事务请求，会对其进行一些预处理，例如创建请求事务头、事务体，会话检查，ACL检查，版本检查等。
  - ProposalRequestProcessor：事务投票处理器。对于非事务请求，会将请求交给CommitProcessor;对于事务请求，除了将请求交给Commitprocessor之外，还会根据请求类型创建对应的Proposal提议，并发送给所有的Follower，发起一次集群内的事务投票。同时，还会将事务请求交给SyncRequestProcessor进行事务日志的记录。
  - SyncRequestProcessor：事务日志记录处理器。主要用来将事务请求记录到事务日志文件中去，同时还会触发zk进行事数据快照。
  - AckRequestProcessor：主要负责在SyncRequestProcessor完成事务日志记录之后，向Proposal的投票收集器发送ack反馈，以通知投票收集器当前服务器已经完成了对改Proposal的事务日志记录。
  - CommitProcessor：事务提交处理器。对于非事务请求，直接交给下一级处理器；对于事务请求，会等待集群内针对Proposal的投票直到该Proposal可被提交。
  - ToBeCommitProcessor：该处理器中有一个ToBeApplied队列，用来储存可被提交的Proposal，将这些请求逐个交给FinalRequestProcessor处理。
  - FinalRequestProcessor：主要用来进行客户端请求返回之前的收尾工作，包括创建客户端请求的响应；针对事务请求，还会将事务应用到内存数据库中。

#### Follower
- 主要职责：
  - 处理客户端非事务请求，转发事务请求给Leader。
  - 参与事务请求Proposal投票
  - 参与Leader选举

#### Observer
- 除了不参与Proposal投票和Leader选举，其他功能和Follower一样。



- ZNode
数据模型中的数据单元，ZNode分为持久节点和临时节点。节点还有一个特殊的属性：SEQUENTIAL，一旦节点被标记上这个属性，便成为有序节点，在这个节点被创建的时候，Zookeeper会自动在其节点名后面追加上一个整型数字，这个整型数字是一个由父节点维护的自增数字。

- Stat
zookeeper会为每个znode维护一个叫做Stat的数据结构，其中记录了znode的3个数据版本，分别是version(当前znode版本),cversion（当前znode子节点版本）,aversion（当前znode的ACL版本）。

- Watcher
事件监听器。在一些特定事件被触发时，服务端将事件通知到感兴趣的客户端。

- ACL
权限控制列表
  - create：创建子节点的权限
  - read：读取节点数据和子节点列表的权限
  - write：更新节点数据的权限
  - delete：删除子节点的权限
  - admin:设置节点ACL的权限

- 为什么最好使用奇数台服务器构成ZooKeeper集群？
> zookeeper容错是指，当宕掉几个zookeeper服务器之后，剩下的个数必须大于宕掉的个数的话整个zookeeper才依然可用。假如我们的集群中有n台zookeeper服务器，那么也就是剩下的服务数必须大于n/2。比如假如我们有3台，那么最大允许宕掉1台zookeeper服务器，如果我们有4台的的时候也同样只允许宕掉1台。所以肯定选择3台较于4台是更优的选择。

### 服务器状态
- LOOKING 不确定Leader状态。该状态下的服务器认为当前集群中没有Leader，会发起Leader选举。
- FOLLOWING 跟随者状态。表明当前服务器角色是Follower，并且它知道Leader是谁。
- LEADING 领导者状态。表明当前服务器角色是Leader，它会维护与Follower间的心跳。
- OBSERVING 观察者状态。表明当前服务器角色是Observer，与Folower唯一的不同在于不参与选举，也不参与集群写操作时的投票。

### zid
- 类似于RDBMS中的事务ID，用于标识一次更新操作的Proposal ID。为了保证顺序性，该zkid必须单调递增。因此ZooKeeper使用一个64位的数来表示，高32位是Leader的epoch，从1开始，每次选出新的Leader，epoch加一。低32位为该epoch内的序号，每次epoch变化，都将低32位的序号重置。这样保证了zkid的全局递增性。

### mutliupdate
- 批量更新也具有原子性，要么都成功，要么都失败。


### 羊群效应
像这样，大量的client都会获得相同的事件通知，而只有很小的一部分client会对 事件通知有响应。我们这里，只有一个client将获得锁，但是所有的client都得到了通知。那么 这就像在网络公路上撒了把钉子，增加了ZooKeeper服务器的压力。

### zookeeper应用场景
#### 数据发布与订阅（配置中心）
- 客户端向zookeeper注册自己想要关注的节点，当节点数据发生变化时，服务端会向客户端发生watcher事件通知，客户端接收到通知，需要向服务端获取最新数据。


#### 负载均衡

#### 命名服务

#### 分布式协调与通知

#### 集群管理

#### master选举
- 使用curator客户端实现Master选举逻辑。用多个线程模拟多个应用，在"/root"持久节点下同时创建相同的临时节点，创建成功的线程作为master节点，创建失败的节点便监听root的子节点变更事件，即被创建的临时节点被删除时会通知不是master的节点去重新选举master。
```java
public class SelectMaster {
    public static void main(String[] args) {
        new Thread(new Master()).start();
        new Thread(new Master()).start();
        while (true){}

    }

    private static class Master implements Runnable{
        private static final String root = "/root";

        @Override
        public void run() {
            CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181",
                    new RetryNTimes(10, 5000));
            client.start();// 连接
            createMasterNode(client);
        }

        public void createMasterNode(CuratorFramework client) {
            try {
                String name = client.create()
                        .withMode(CreateMode.EPHEMERAL)
                        .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
                        .forPath(root + "/binding");
                if (!StringUtils.isEmpty(name)) {
                    System.out.println(name);
                    System.out.println("i am master."+ Thread.currentThread().getName());
                }
            } catch (Exception e) {
                System.out.println("create failed. i am not master." + Thread.currentThread().getName());
            }
            watch(client);
        }

        public void watch(CuratorFramework client) {
            System.out.println("注册子节点变更事件。" + Thread.currentThread().getName());
            // 注册子节点变更监听
            try {
                client.getChildren().usingWatcher((CuratorWatcher) event -> {
                    System.out.println("监控： " + event);
                    switch (event.getType()) {
                        case NodeChildrenChanged:
                            createMasterNode(client);
                            break;
                        default:break;
                    }
                }).forPath(root);
            } catch (Exception e) {
                System.out.println("根节点不存在。注册监听事件失败。");
            }
        }
    }
}
```

#### 分布式锁

#### 分布式队列




