# LockSupport剖析

LockSupport通过许可（permit）实现线程挂起、挂起线程唤醒功能。permit可以理解为每个已启动线程维持的一个int类型状态位counter。线程分别通过执行LockSupport静态方法park\(\)、unPark\(\)方法来完成挂起、唤醒操作。

park\(\)执行过程：

判断当前线程是否发生中断：

* 中断：线程继续执行。
* 未中断：判断counter值。counter值1，表示线程拥有permit，将状态位置为0，线程继续执行；counter值0，表示线程没有permit，线程阻塞。

unPark\(\)执行过程：

（1）记录counter当前状态，将counter置为1（不管状态位值为啥）。

（2）根据当前线程状态位信息，判断unPark\(\)对应线程是否挂起。

* 挂起：执行线程唤醒操作，唤醒后将counter置为0；
* 未挂起：返回。





