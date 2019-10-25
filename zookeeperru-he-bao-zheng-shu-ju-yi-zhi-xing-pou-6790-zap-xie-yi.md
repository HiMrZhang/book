# **zookeeper如何保证数据一致性剖析**

ZooKeeper服务有两种不同的运行模式。一种是**"独立模式"**\(standalone mode\)，即只有一个ZooKeeper服务器。这种模式较为简单，比较适合于单元测试、测试环境中采用，但是不能保证高可用性和恢复性。在生产环境中的ZooKeeper通常以**"复制模式"**\(replicated mode\)运行于一个计算机集群上，这个计算机集群被称为一个"集合体"\(ensemble\)。保证ZooKeeper服务的高可用性需要**"复制模式"**，通过写多份来冗余数据，那么如何解决写多份带来一致性问题呢？如何解决一致性问题又会带来性能问题？

### 数据一致性需满足哪些特性呢？

1. **顺序一致性 **如果clientA将zookeeper集群中节点A的数据更新为a，之后又将节点A的数据更新为b，期间没有其它客户端对节点A进行更新操作，需保证其它客户端在看到节点A的数据为b时，之后不会看到节点A的数据为a。
2. **原子性 **如果clientA更新zookeeper集群中节点A的数据失败，则不会有客户端会看到这个更新后的结果。
3. **单一系统镜像 **如果client A在同一个会话中连接到一台新的节点，要保证新节点的数据不能滞后于故障节点数据**（zookeeper集群中任意节点可提供读服务）**。
4. **持久性 **如果一个更新一旦成功，其结果就会持久存在并且不会被撤销。不会受到服务器故障的影响。

### **数据一致性保障机制-ZAP协议**

Zookeeper 是通过 Zab （Zookeeper Atomic Broadcast）协议来保证分布式事务的最终一致性。Zab协议有两种模式，它们分别是恢复模式和广播模式。

* **广播模式**

广播模式类似一个简单的两阶段提交：Leader发起一个请求，收集选票，并且最终提交。两段提交要求协调者必须等到所有的参与者全部反馈ACK确认消息后，再发送commit消息。而Zab协议中 Leader 只要半数以上的Follower成功反馈ACK确认即可，不需要收到全部Follower反馈ACK确认。

（1）客户端向leader节点发起写请求操作**（写操作必须通过leader节点完成）**。

（2）Leader 服务器将客户端的请求转化为事务 Proposal 提案，每个 Proposal 都有一个全局单调递增的ID，即zxid。

广播协议在所有的通讯过程中使用TCP的FIFO信道，通过使用该信道，使保持有序性变得非常的容易。通过FIFO信道，消息被有序的deliver。只要收到的消息一被处理，其顺序就会被保存下来。

Leader会广播已经被deliver的Proposal消息。在发出一个Proposal消息前，Leader会分配给Proposal一个单调递增的唯一id，称之为zxid。因为Zab保证了因果有序， 所以递交的消息也会按照zxid进行排序。广播是把Proposal封装到消息当中，并添加到指向Follower的输出队列中，通过FIFO信道发送到 Follower。当Follower收到一个Proposal时，会将其写入到磁盘，可以的话进行批量写入。一旦被写入到磁盘媒介当 中，Follower就会发送一个ACK给Leader。 当Leader收到了指定数量的ACK时，Leader将广播commit消息并在本地deliver该消息。当收到Leader发来commit消息 时，Follower也会递交该消息。

需要注意的是， 该简化的两阶段提交自身并不能解决Leader故障，所以我们 添加恢复模式来解决Leader故障。

* **恢复模式**



