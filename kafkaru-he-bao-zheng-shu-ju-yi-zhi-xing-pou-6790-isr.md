# kafka如何保证数据一致性剖析

众所周知，Kafka是一款高性能、高可用的分布式发布订阅消息系统， Apache 旗下的一个子项目，目前已成为开源领域应用最广泛的消息系统之一。 从 0.10 版本开始，Kafka Streams java库提供了流处理数据的基本操作，从此Kafka 的标语已经从“一个高吞吐量，分布式的消息系统”改为"一个分布式流平台"。保证Kafka高可用模型依靠的是副本机制，副本机制保障机器宕机不会发生数据丢失问题。那么如何解决写多份带来一致性问题呢？如何解决一致性问题又会带来性能问题？

### 数据一致性需满足什么条件呢？

若某条消息对client可见，那么即使Leader挂了，在新Leader上数据依然可见。

### **数据一致性保障机制-ISR机制**

ISR \(In-Sync Replicas\)是Leader在Zookeeper（/brokers/topics/\[topic\]/partitions/\[partition\]/state）目录中动态维护基本保持同步的Replica列表，该列表中保存的是与Leader副本保持消息同步的所有副本对应的节点id。如果一个Follower宕机或者其落后情况超过任意参数replica.lag.time.max.ms（延迟时间）、replica.lag.max.messages（延迟条数）设置阈值，则该Follower副本节点将从ISR列表中移除。

