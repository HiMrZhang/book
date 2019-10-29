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

![](/assets/yarn-2.png)

从上图可知，yarn框架中移除了JobTracker 和 TaskTracker ，取而代之的是 ResourceManager, ApplicationMaster 与 NodeManager 三个部分。Yarn 框架相对于老的 MapReduce 框架什么优势呢？

1. 这个设计大大减小了 JobTracker（也就是现在的 ResourceManager）的资源消耗，并且让监测每一个 Job 子任务 \(tasks\) 状态的程序分布式化了，更安全、更优美。
2. 在新的 Yarn 中，ApplicationMaster 是一个可变更的部分，用户可以对不同的编程模型写自己的 AppMst，让更多类型的编程模型能够跑在 Hadoop 集群中，可以参考 hadoop Yarn 官方配置模板中的 mapred-site.xml 配置。
3. 对于资源的表示以内存为单位 \( 在目前版本的 Yarn 中，没有考虑 cpu 的占用 \)，比之前以剩余 slot 数目更合理。
4. 老的框架中，JobTracker 一个很大的负担就是监控 job 下的 tasks 的运行状况，现在，这个部分就扔给 ApplicationMaster 做了，而 ResourceManager 中有一个模块叫做 ApplicationsMasters\( 注意不是 ApplicationMaster\)，它是监测 ApplicationMaster 的运行状况，如果出问题，会将其在其他机器上重启。
5. Container 是 Yarn 为了将来作资源隔离而提出的一个框架。这一点应该借鉴了 Mesos 的工作，目前是一个框架，仅仅提供 java 虚拟机内存的隔离 ,hadoop 团队的设计思路应该后续能支持更多的资源调度和控制 , 既然资源表示成内存量，那就没有了之前的 map slot/reduce slot 分开造成集群资源闲置的尴尬情况。

## Yarn应用提交过程

![](/assets/yarn-3.png)

1. 客户端程序向 ResourceManager 提交应用并请求一个 ApplicationMaster 实例；
2. ResourceManager 找到一个可以运行一个 Container 的 NodeManager，并在这个 Container 中启动 ApplicationMaster 实例。
3. ApplicationMaster 向 ResourceManager 进行注册，注册之后提交应用的客户端就可以查询 ResourceManager 获得自己 ApplicationMaster 的详细信息，与ApplicationMaster 直接交互（客户端主动和 ApplicationMaster 交流，向 ApplicationMaster 发送一个满足自己需求的资源请求）；
4. 在平常的操作过程中，ApplicationMaster 根据 resource-request协议 向 ResourceManager 发送 resource-request请求；
5. 当 Container 被成功分配后，ApplicationMaster 通过向 NodeManager 发送 container-launch-specification信息 来启动Container，container-launch-specification信息包含了能够让Container 和 ApplicationMaster 交流所需要的资料；
6. 应用程序的代码以 task 形式在启动的 Container 中运行，并把运行的进度、状态等信息通过 application-specific协议 发送给ApplicationMaster；
7. 在应用程序运行期间，提交应用的客户端主动和 ApplicationMaster 交流获得应用的运行状态、进度更新等信息，交流协议也是 application-specific协议；
8. 应用程序执行完成并且所有相关工作也已经完成，ApplicationMaster 向 ResourceManager 取消注册然后关闭，用到所有的 Container 也归还给系统。



