# elasticsearch如何保证数据一致性原理剖析

ElasticSearch是建立在全文搜索引擎 Apache Lucene\(TM\) 基础上的分布式，高性能、高可用、可伸缩的实时搜索和分析引擎，可支持扩展到上百台服务器，处理PB级别的结构化或非结构化数据。ElasticSerarch通过分片与副本机制来保障集群的高性能、高可靠。ElasticSearch如何保证副本之间数据一致性呢？

## ElasticSearch集群数据同步机制：

![](/assets/es-1.png)

Elasticsearch采用一主多副复制方式，没有采用节点级别的主从复制，而是基于分片。

Index由多个Shard组成，每个Shard有一个主节点和多个副本节点。数据写入的时候，首先根据\_routing规则对路由参数（路由参数默认使用\_id，Index Request中可以设置使用哪个Filed的值作为路由参数）进行Hash确定要写入的Shard，最后从集群的Meta中找出该Shard的Primary节点写入数据，Primary Shard成功写入数据后，Primary Shard将请求同时发送给多个Replica Shard（InSyncAllocationIds），请求在全部Replica Shard上执行成功并响应Primary Shard后，Primary Shard返回结果给客户端。

该模式，Primary要等所有Replica返回才能返回给客户端，那么延迟就会受到最慢的Replica的影响，写入操作的延时就等于latency = Latency\(Primary Write\) + Max\(Replicas Write\)。副本的存在，降低了写入效率会较低，但是可以避免单机或磁盘故障导致数据丢失。

Replica写入失败，Primary会执行一些重试逻辑，尽可能保障Replica中写入成功。如果一个Replica最终写入失败，Primary会将Replica节点报告给Master，然后Master更新Meta中Index的InSyncAllocations配置，将Replica从中移除，移除后它就不再承担读请求。在Meta更新到各个Node之前，用户可能还会读到这个Replica的数据，但是更新了Meta之后就不会了。所以这个方案并不是非常的严格，考虑到ES本身就是一个近实时系统，数据写入后需要refresh才可见，所以一般情况下，在短期内读到旧数据应该也是可接受的。

设置wait\_for\_active\_shards参数（可在Index的setting、请求中设置）可保证Es集群数据一致性更高的可能性。该参数设置每次写入shard至少具有的active副本数，如果副本数小于参数值，此时不允许写入。

## ElasticSearch节点数据写入机制：

![](/assets/es-2.png)由于Lucene的索引读写的隔离性，需要在添文档后对IndexWriter进行commit，在搜索的时候IndexReader需要重新的打开，这样才能保证查询结果的实时性，然而当硬盘上的索引非常大的时候，IndexWriter的commit操作和IndexReader的open操作都是非常慢的，根本达不到实时性的需要。请求到达DataNode后，首先经过Lucene引擎生成document存入到indexing buffer区中，然后通过refresh操作（默认情况下，es集群中的每个shard会每隔1秒自动refresh一次，这是es是近实时的搜索引擎而不是实时的原因）将indexing buffer区中新增Document在filesystem cache中生成segment，这个sengment就可以打开和查询，从而确保在可见性，refresh操作是非常轻量级的，相对耗时较少，之后经过一定的间隔或外部触发后segment才会被flush到磁盘上，这个操作非常耗时。flush操作完成后会清空掉旧的TransLog。

为了避免segement从内存flush到磁盘这段时间节点crash掉，导致数据丢失，数据经过lucene引擎处理后会写入TransLog，当节点重启后会根据TransLog恢复数据到Checkpoint。通过设置TransLog的Flush频率可以控制可靠性，要么是按请求，每次请求都Flush；要么是按时间，每隔一段时间Flush一次。一般为了性能考虑，会设置为每隔5秒或者1分钟Flush一次，Flush间隔时间越长，可靠性就会越低。对于GetById查询，可以直接从TransLog中获取数据，这时候就成了RT（Real Time）实时系统。

### **数据一致性保障**

要保证数据写入到ElasticSerach是安全的，高可靠的，需要如下的配置：

* 设置wait\_for\_active\_shards参数大于等于2。

* 设置TransLog的Flush策略为每个请求都要Flush。



