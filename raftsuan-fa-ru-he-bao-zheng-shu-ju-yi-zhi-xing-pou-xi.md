# Raft算法如何保证数据一致性剖析

共识算法Raft通过集群中唯一的领导者管理集群其他服务器上的日志复制来保证数据一致性。共识问题在Raft中被分解为leader选举、log复制两个相对独立的子问题。

## 一致性问题

在分布式系统中,一致性问题\(consensus problem\)是指对于一组服务器，给定一组操作，我们需要一个协议使得最后它们的结果达成一致。

## leader选举机制

Raft集群中每个节点有三种状态：Follower，Candidate，Leader，状态之间是互相转换。节点启动的初始状态为follower，每个Follower节点上都有一个倒计时器 （随机在 150ms 到 300ms 之间设置Election Timeout时间），倒计时截止后状态转换为Candidate，并开始发起leader选举。Follower节点通过接受leader hearteat或者candidate RequestVote请求重设Election Timeout来维持Follower状态。

1. Follower节点定时器截止后增加当前的term（一段任意的时间序号，相当于一个国家的朝代，每个Term以选举开始，如果选举成功，则集群会在当前Term下由当前的leader管理），节点状态转换为Candidate。
2. Candidate节点首先增加投票计数器，投票给自己作为新的领导者，然后向集群中的其他服务器发送RequestVote RPC请求。
3. 收到RequestVote的服务器，在同一term中只会按照先到先得投票给一个与自身log一样或者更（gèng）新的candidate节点。
4. Candidate 节点获得超过一半的节点投支持票，该节点状态将转换为leader。其它Candidate 节点收到term值等于或大于当前自身term值的leader hearteat后，该节点状态将转换为follower。如果Candidate 节点定时器截止，仍没有选出leader，将由最先截止的Candidate 节点发起下一轮投票（解决多个Candidate同时获取相同选票无法确定leader问题）。

随机设置Election Timeout时间，可避免多个follower同时变为Candidate状态，发起leader投票。

## 日志复制机制

Raft集群采用master/slave结构，leader为master，follower为slave，只有leader能够接收client请求。

1. leader节点接收客户端的请求命令转换为操作记录并写入日志。
2. leader节点向其他的服务器发送AppendEntries RPCs请求。
3. follower节点接收到leader的AppendEntries RPCs请求后，将操作日志写入日志并返回结果给leader节点。
4. leader节点收到半数以上的节点回应该条记录已写入日志，则可认为该记录是有效的，leader节点将该记录提交，并将执行结果返回给客户端。leader节点在下次心跳AppendEntries RPCs请求中，会记录可被提交的日志条目编号commitIndex。
5. follower节点在收到leader的AppendEntries RPCs请求后，会根据请求中的commitIndex，将本地日志中commitindex之前未提交记录提交。

## 案例分析

![](/assets/rsa-1.png)

客户端依次向leader写入x=3、y=2、c=1，节点日志复制情况如上图所示。![](/assets/rsa-2.png)

server1 crash掉，由于server2中log更（gèng）新，server2被选举为leader。y=2、x=3数据处于未提交状态，Client 不会收到 Ack 超时失败后可发起重试在新leader中重新提交。

![](/assets/rsa-3.png)leader将y=2同步至follower并提交，同时接收新请求d=4。y=2操作在写入失败后可发起重试操作，针对这种情况 Raft通过内部去重机制实现幂等性来保证一致性。

![](/assets/rsa-4.png)server1恢复后，将未提交消息清除掉，然后在完成log同步。

