# elasticsearch如何保证数据一致性原理剖析

ElasticSearch是建立在全文搜索引擎 Apache Lucene\(TM\) 基础上的分布式，高性能、高可用、可伸缩的实时搜索和分析引擎，可支持扩展到上百台服务器，处理PB级别的结构化或非结构化数据。ElasticSerarch通过分片与副本机制来保障集群的高性能、高可靠。ElasticSearch如何保证副本之间数据一致性呢？

## ElasticSearch数据同步机制：





