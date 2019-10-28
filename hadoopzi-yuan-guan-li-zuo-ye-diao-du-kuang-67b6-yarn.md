# Hadoop资源管理与作业调度框架-yarn

提到Hadoop，大家可能首先想到的是Hdfs存储、mapreduce离线计算，Hadoop2.x推出yarn（Yet Another Resource Negotiator）之后，hadoop已摇身一变为资源管理与作业调度平台，基于yarn可在hadoop集群上可运行mepreduce（离线计算）、spark（内存计算）、strom（流计算）、MPI等计算框架， 从而将hadoop推向了另一个高度。

## yarn框架组成

![](/assets/yarn-1.png)

Yarn采用master/slave结构，Resource Manger进程所在节点为master，Node Manager所在节点为slave。YARN 由 ResourceManager、NodeManager、ApplicationMaster 和 Container 等主要组件构成。

* ResourceManager是Master上一个独立运行的进程，负责整个系统的资源管理和分配，包括处理客户端请求、启动/监控 ApplicationMaster、监控 NodeManager、资源的分配与调度。

* NodeManager是Slave上一个独立运行的进程，负责上报进程所在节点的状态。
* Container是slave中运行的进程，包涵内存、CPU等资源，YARN以Container作为资源分配的一个单位。ApplicationMaser、task在Container中运行。

* ApplicationMaster负责向Yarn ResourceManager申请资源执行Task，并负责监控、管理这个Application的所有Task在cluster中各个节点上的具体运行。不同的计算框架有不同的ApplicationMaster实现。

## MapreduceV1与MapreduceV2（yarn）对比







