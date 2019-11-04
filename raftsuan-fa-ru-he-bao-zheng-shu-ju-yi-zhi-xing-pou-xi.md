# Raft算法如何保证数据一致性剖析

共识算法Raft通过集群中唯一的领导者管理集群其他服务器上的日志复制来保证数据一致性。共识问题在Raft中被分解为leader选举、log复制两个相对独立的子问题。

## leader选举机制

Raft集群中每个节点有三种状态：Follower，Candidate，Leader，状态之间是互相转换。节点启动的初始状态为follower，leader节点不断的向集群中所有follower节点发送heartbeat信息来确保集群状态，每个Follower节点上都有一个倒计时器 \(Election Timeout\)，时间随机在 150ms 到 300ms 之间，每次收到leader解决heartbeat信息都会重设Election Timeout开始倒计时。Follower节点中倒计时截止后转换为Candidate状态，由Candidate发起选举。

1. Candidate节点首先增加投票计数器，投票给自己作为新的领导者。



