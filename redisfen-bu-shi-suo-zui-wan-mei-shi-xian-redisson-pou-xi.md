# Redis分布式锁最完美实现Redisson剖析

同一服务在单实例部署环境可通过synchronized、ReentrantLock本地锁来保证在一次只能允许一个线程执行被锁住的代码块，但在分布式多实例部署环境需实现分布式锁来保证互斥性。redission堪称redis分布式锁最完美实现，接下来一起来剖析redisson实现机制。

## 实现分布式锁需要关注哪些细节呢？

* 确保互斥：在同一时刻，必须保证锁至多只能被一个客户端持有。
* 不能死锁：在一个客户端在持有锁的期间崩溃而没有主动解锁情况下，也能保证后续其他客户端能加锁。
* 避免活锁：在获取锁失败的情况下，反复进行重试操作，占用Cpu资源，影响性能。
* 实现更多锁特性：锁中断、锁重入、锁超时等。
* 确保客户端只能解锁自己持有的锁。

## 源码地址 {#Redis实现之Redisson原理}

[https://github.com/redisson/redisson](https://github.com/redisson/redisson)

## 加锁：

![](/assets/redission-1.png)

org.redisson.RedissonLock类tryLockInnerAsync通过eval命令执行Lua代码完成加锁操作。KEYS\[1\]为锁在redis中的key，key对应value为map结构，ARGV\[1\]为锁超时时间，ARGV\[2\]为锁value中的key。ARGV\[2\]由**UUID+threadId** 组成，用来标记锁被谁持有。

\(1\) 第一个If判断key是否存在，不存在完成加锁操作

* redis.call\('hset', KEYS\[1\], ARGV\[2\], 1\);创建key\[1\] map中添加key：ARGV\[2\] ，value：1
* redis.call\('pexpire', KEYS\[1\], ARGV\[1\]\);设置key\[1\]过期时间，避免发生死锁。

eval命令执行Lua代码的时候，Lua代码将被当成一个命令去执行，并且直到eval命令执行完成，Redis才会执行其他命令。可避免第一条命令执行成功第二条命令执行失败导致死锁。

\(2\)第二个if判断key存在且当前线程已经持有锁, 重入:

* redis.call\('hexists', KEYS\[1\], ARGV\[2\]\)；判断redis中锁的标记值是否与当前请求的标记值相同，相同代表该线程已经获取锁。

* redis.call\('hincrby', KEYS\[1\], ARGV\[2\], 1\);记录同一线程持有锁之后累计加锁次数实现锁重入。

* redis.call\('pexpire', KEYS\[1\], ARGV\[1\]\); 重制锁超时时间。

\(3\)key存在被其他线程获取的锁, 等待:

* redis.call\('pttl', KEYS\[1\]\);加锁失败返回锁过期时间。

## 锁等待

![](/assets/redission-2.png)

（1）步骤一：调用加锁操作；

（2）步骤二：步骤一中加锁操作失败，订阅消息，利用redis的pubsub提供一个通知机制来减少不断的重试，避免发生活锁。

（3）步骤三：

![](/assets/redission-3.png)getLath\(\)获取RedissionLockEntry实例latch变量，由于permits为0，所以嗲用acquire\(\)方法后线程阻塞。

## 解锁

![](/assets/redission-4.png)

（1）第一个if判断锁对应key是否存在，不存在publish消息，将获取锁被阻塞的线程恢复重新获取锁；

（2）第二个if判断锁对应key存在，value中是否存在当前要释放锁的标示，不存在返回nil，确保锁只能被持有的线程释放。

（3）对应key存在，value中存在当前要释放锁的标示，将锁标示对应值-1，第三个if判断锁标示对应的值是否大于0，大于0，表示有锁重入情况发生，重新设置锁过期时间。

（4）对应key存在，value中存在当前要释放锁的标示，将锁标示对应值-1后等于0，调用del操作释放锁，并publish消息，将获取锁被阻塞的线程恢复重新获取锁；

![](/assets/redission-5.png)订阅者接收到publish消息后，执行release操作，调用acquire被阻塞的线程将继续执行获取锁操作。

