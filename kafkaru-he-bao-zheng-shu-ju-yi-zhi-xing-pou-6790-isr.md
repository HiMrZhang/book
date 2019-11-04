# kafka如何保证数据一致性剖析

众所周知，Kafka是一款高性能、高可用的分布式发布订阅消息系统， Apache 旗下的一个子项目，目前已成为开源领域应用最广泛的消息系统之一。 从 0.10 版本开始，Kafka Streams java库提供了流处理数据的基本操作，从此Kafka 的标语已经从“一个高吞吐量，分布式的消息系统”改为"一个分布式流平台"。保证Kafka高可用模型依靠的是副本机制，副本机制保障机器宕机不会发生数据丢失问题。那么如何解决写多份带来一致性问题呢？如何解决一致性问题又会带来性能问题？

### 数据一致性需满足什么条件呢？

由于kafka采用At least once（消息可能会重复传输但绝不会丢）传输保障，所以数据一致性必须符合以下条件。

若某条消息对client可见，那么即使Leader挂了，在新Leader上数据依然可见。

### **ISR复制机制**

ISR \(In-Sync Replicas\)是Leader在Zookeeper（/brokers/topics/\[topic\]/partitions/\[partition\]/state）目录中动态维护基本保持同步的Replica列表，该列表中保存的是与Leader副本保持消息同步的所有副本对应的节点id。如果一个Follower宕机或者其落后情况超过任意参数replica.lag.time.max.ms（延迟时间）、replica.lag.max.messages（延迟条数，Kafka 0.10.x版本后移除）设置阈值，则该Follower副本节点将从ISR列表中剔除并存入OSR\(Outof-Sync Replicas\)列表。

ISR冗余备份机制核心逻辑围绕HW值、LEO值展开，接下来了解一下HW、LEO相关概念。

LEO（last end offset）日志末端偏移量，记录了该副本对象底层日志文件中下一条消息的位移值。

HW（highwatermark），高水印值，任何一个副本对象的HW值一定不大于其LEO值，而小于或等于HW值的所有消息被认为是“已提交的”或“已备份的”。consumer只能消费已提交的消息，HW之后的数据对consumer不可见。

![](/assets/isr-1.png)数据同步过程如下：

1. Follower向Leader发送fetch请求（此过程类似于普通Customer，区别在于内部broker的读取请求，没有HW的限制）。
2. Leader接收到Follwer fetch操作后根据fetch请求中Postion从自身log中获取相应数据，并根据fetch请求中Postion更新leader中存储的follower LEO。通过follower LEO读取存在于ISR列表中副本的LEO（包括leader自己的LEO）值，并选择最小的LEO值作为HW值。
3. Follower接收到leader的数据响应后，开始向底层log写数据，每当新写入一条消息，其LEO值就会加1，写完数据后，通过比较当前LEO值与FETCH响应中leader的HW值，取两者的小者作为新的HW值。

由此可见，Kafka复制机制既不是完全的同步复制，也不是单纯的异步复制，Kafka通过 ISR复制机制在保障数据一致性情况下又可提供高吞吐量。

### **数据一致性保障**

request.required.acks：该参数在producer向leader发送数据时设置。

* 0：producer无需等待来自broker的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。

* 1（默认）：producer在ISR中的leader已成功收到数据并得到确认。如果leader宕机了，则会丢失数据。

* -1：producer需要等待ISR中的所有follower都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当ISR中只剩下一个leader时，这样就变成了acks=1的情况。

min.insync.replicas：该参数在broker或者topic层面进行设置，设定ISR中的最小副本数是多少，默认值为1，当且仅当request.required.acks参数设置为-1时，此参数才生效。如果ISR中的副本数少于min.insync.replicas配置的数量时，客户端会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。

unclean.leader.election.enable：

* true：默认值，所有replica都有成为leader的可能。
* false：只有在ISR中存在的replica才有成为leader的可能。

要保证数据写入到Kafka是安全的，高可靠的，需要如下的配置：

* topic的配置：replication.factor&gt;=3,即副本数至少是3个；2&lt;=min.insync.replicas&lt;=replication.factor

* broker的配置：leader的选举条件unclean.leader.election.enable=false

* producer的配置：request.required.acks=-1\(all\)，producer.type=sync

### 案例分析

![](/assets/isr-2.png)如上图所示，该分区ISR列表值为Broker1、Broker2、Broker3 ID值{1,2,3}，broker1为leader，broker2、broker3为Follower。Producer向leader发送消息m1、m2、m3。ISR中所有副本都已同步m1，HW指向m2起始位置。![](/assets/isr-3.png)此时Broker1 crash掉，Broker1 ID从ISR列表中移除，该分区ISR列表值为Broker2、Broker3 ID值{2,3}Broker2选为leader，Broker3为Follower。新的leader会发送“指令”让其余的follower截断至自身的HW的位置然后再拉取新的消息。![](/assets/isr-4.png)Producer向leader发送消息m4、m5。ISR中所有副本都已同步m5，HW指向m6起始位置。

![](/assets/isr-5.png)此时Broker1恢复，将自己的log文件截断到上次checkpointed时刻的HW的位置，然后再从leader中同步消息。完成同步后，加入ISR列表，最终该分区ISR列表值为Broker1、Broker2、Broker3 ID值{1,2,3}。

