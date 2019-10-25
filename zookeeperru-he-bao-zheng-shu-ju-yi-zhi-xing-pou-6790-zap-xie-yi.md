# **zookeeper如何保证数据一致性剖析**

ZooKeeper服务有两种不同的运行模式。一种是**"独立模式"**\(standalone mode\)，即只有一个ZooKeeper服务器。这种模式较为简单，比较适合于单元测试、测试环境中采用，但是不能保证高可用性和恢复性。在生产环境中的ZooKeeper通常以**"复制模式"**\(replicated mode\)运行于一个计算机集群上，这个计算机集群被称为一个"集合体"\(ensemble\)。保证ZooKeeper服务的高可用性需要**"复制模式"**，通过写多份来冗余数据，那么如何解决写多份带来一致性问题呢？如何解决一致性问题又会带来性能问题？

### 数据一致性需满足哪些特性呢？

1. **顺序一致性 **如果clientA将zookeeper集群中节点A的数据更新为a，之后又将节点A的数据更新为b，期间没有其它客户端对节点A进行更新操作，需保证其它客户端在看到节点A的数据为b时，之后不会看到节点A的数据为a。
2. **原子性 **如果clientA更新zookeeper集群中节点A的数据失败，则不会有客户端会看到这个更新后的结果。
3. **单一系统镜像 **如果client A在同一个会话中连接到一台新的节点，要保证新节点的数据不能滞后于故障节点数据**（zookeeper集群中任意节点可提供读服务）**。
4. **持久性 **如果一个更新一旦成功，其结果就会持久存在并且不会被撤销。不会受到服务器故障的影响。

### **数据一致性保障机制-ZAB协议**

Zookeeper 是通过 Zab （Zookeeper Atomic Broadcast）协议来保证分布式事务的最终一致性。Zab协议有两种模式，它们分别是恢复模式和广播模式。

#### 一、**广播模式**

广播模式类似一个简单的两阶段提交：Leader发起一个请求，收集选票，并且最终提交。两段提交要求协调者必须等到所有的参与者全部反馈ACK确认消息后，再发送commit消息。而Zab协议中 Leader 只要半数以上的Follower成功反馈ACK确认即可，不需要收到全部Follower反馈ACK确认。![](/assets/1.png)

（1）客户端向leader节点发起写请求操作**（写操作必须通过leader节点完成）**。

（2）Leader 节点收到客户端写请求后生成事务 Proposal 消息，在广播消息前，leader节点会给每个 Proposal分配一个全局单调递增的ID，即zxid。ZAB保证了因果有序， 所有递交的消息也会按照zxid进行排序。

（3）Leader 节点与每个 Follower 节点通讯过程使用TCP的FIFO信道，将需要广播的 Proposal 依次放到FIFO信道中，通过FIFO信道，消息被有序的deliver，从而消息保证有序性。

（4）Follower 节点收到一个 Proposal 后，会首先将其以事务日志的方式写入本地磁盘中，写入成功后，Follower节点会向 Leader 发送一个 Ack 响应消息。

（5）Leader 节点接收到超过半数以上 Follower 节点的 Ack 响应消息后，leader将广播 commit 消息。

（6）Leader 节点向所有 Follower节点 广播 commit 消息同时，自身也会完成事务提交。Follower节点 接收到 commit 消息后，会将上一条事务提交。

需要注意的是， 该简化的两阶段提交自身并不能解决Leader故障，所以我们 添加恢复模式来解决Leader故障。

#### 二、**恢复模式**

Leader 服务器出现崩溃或者由于网络原因导致 Leader 服务器失去了与过半 Follower 的联系，那么就会进入崩溃恢复模式。为了保证进度，恢复过程中必须选举出一个新Leader，并且最终让所有的Server拥有一个正确的状态。恢复模式主要包括两部分：Leader选举 和 数据恢复

**恢复阶段保证**

* 若一条消息在一台机器上被deliver，那么该消息必须将在每台机器上deliver。即使那台机器故障了。
* 一条被skip的消息，仍然需要被skip。

**案例分析**

![](/assets/2.png)

如上图所示，Server1是一个Leader，我们用Leader1表示，Server2和Server3为Follower。首先Leader发起了两个Proposal，P1和P2，并将P1、P2发送给了Server1和Server2。然后Leader对P1发起了Commit即C1，之后又发起了一个Proposal即P3，再后来又对P2发起了commit即C2，就在此时我们的Leader挂了。那么这时候，C1、P3和C2这两个消息只有Leader自己收到了。

![](/assets/3.png)Server1挂掉以后，Server3被选举为Leader，用Leader3表示。在接收请求之前，Leader3首先检查事物日志中是否存在未提交、且在超过半数的节点中存在的消息P1、P2，并依次发出C1、C2分别将P1、P2提交，之后Leader3才可发出了新的提议P10000001和C10000001。



