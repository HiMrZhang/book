# zookeeper分布式锁最完美实现Curator剖析

zookeeper分布式锁基于zookeeper中有序节点实现分布有序等待锁队列，通过watch机制监听锁释放，大大提升强锁效率，避免自旋等锁，浪费cpu资源，同时只watch前一个节点可避免锁释放之后通知所有节点重新竞争锁导致的羊群效应。

curator基于zookeeper实现了InterProcessMutex（分布式可重入排它锁）、InterProcessSemaphoreMutex（分布式排它锁）、InterProcessReadWriteLock（分布式读写锁）、InterProcessMultiLock（将多个锁作为单个实体管理的容器）四种不同的锁实现，本文将介绍InterProcessMutex的实现原理。

## 实现步骤

（1）在根节点（持久化节点，eg：/lock）下创建临时有序节点。

（2）获取根节点下所有子节点，判断上一步中创建的临时节点是否是最小的节点。

* 是：加锁成功
* 不是：加锁失败，对前一个节点加watch，收到删除消息后，加锁成功。

（3）执行被锁住的代码块。

（4）删除自己在根节点下创建的节点，解锁成功。

## 源码位置

[https://github.com/apache/curator](https://github.com/apache/curator)

## 加锁

![](/assets/curator-1.png)

InterProcessMutex类acquire\(\)方法通过调用internalLock\(\)方法完成加锁操作。

（1）首先判断该线程是否已经获取锁，如果已经获取锁lockcount加一，加锁成功。该步骤支持锁重入。

（2）调用attemptlock\(\)方法获取锁。

（3）获取锁成功后向threadData中添加数据，记录该线程已经获取锁。

继续分析attemptlock\(\)方法  
![](/assets/curator-2.png)

（1）ourPath = driver.createsTheLock\(client, path, localLockNodeBytes\);创建临时有序节点。

（2）hasTheLock = internalLockLoop\(startMillis, millisToWait, ourPath\);判断上一步中创建的临时节点是否是根结点下最小的节点。如果是返回true，加锁成功。

继续分析internalLockLoop\(\)方法

![](/assets/curator-3.png)（1）进入循环，获取根结点下面所有子节点，判断上一步中创建的临时节点是否是根结点下最小的节点。如果是直接返回。

（2）对当前节点前一个节点加watch。

（3）调用wait\(\)方法阻塞。

## 解锁

![](/assets/curator-4.png)InterProcessMutex类release\(\)方法完成解锁操作。

（1）判断是否存在锁重入的情况，如果存在锁重入，lockcount减一，如果lockcount大于0直接返回；

（2）调用releaseLock\(\)方法完成解锁操作。

继续继续分析releaseLock\(\)方法。

![](/assets/curator-5.png)releaseLock\(\)调用deleteOurPath\(\)方法将临时节点删除。删除后watch触发，调用notifyFromWatcher\(\)方法执行notifyAll\(\)方法，将线程唤醒继续执行操作。

## 总结

zookeeper集群使用zap协议来保证集群之间数据的一致性，频繁的进行写、监听操作将对zk集群产生较大压力，所以不推荐大家使用。

