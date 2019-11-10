# LockSupport剖析

LockSupport通过许可（permit）实现线程挂起、挂起线程唤醒功能。permit可以理解为每个已启动线程维持的一个int类型状态位counter。线程分别通过执行LockSupport静态方法park\(\)、unPark\(\)方法来完成挂起、唤醒操作。

park\(\)执行过程：

（1）判断当前线程是否发生中断：

* 中断：线程继续执行。
* 未中断：判断counter值。counter值1，表示线程拥有permit，将状态位置为0，线程继续执行；counter值0，表示线程没有permit，线程阻塞。

unPark\(\)执行过程：

（1）记录counter当前状态，将counter置为1（不管状态位值为啥）。

（2）根据当前线程状态位信息，判断unPark\(\)对应线程是否挂起。

* 挂起：执行线程唤醒操作，唤醒后将counter置为0；
* 未挂起：返回。

## Tips

（1）线程启动后unPark\(\)在park\(\)操作前执行仍可继续运行。

```
package com.easysoft;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.locks.LockSupport;

@Slf4j
public class LockSupportDemo {


    public static class TaskThread extends Thread {
        public TaskThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            try {
                Thread.sleep(3 * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.info("in thread[{}]  ", getName());
            log.info("park thread[{}]  ", getName());
            LockSupport.park(this);
            log.info("do something...");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        TaskThread t1 = new TaskThread("task-1");
        t1.start();
        LockSupport.unpark(t1);
        log.info("unpark thread[{}]  ", t1.getName());

        t1.join();
    }
}
```

输出结果：

```
2019-11-10 17:39:44.675  [main] INFO  com.easysoft.LockSupportDemo -unpark thread[task-1]  
2019-11-10 17:39:47.676  [task-1] INFO  com.easysoft.LockSupportDemo -in thread[task-1]  
2019-11-10 17:39:47.676  [task-1] INFO  com.easysoft.LockSupportDemo -park thread[task-1]  
2019-11-10 17:39:47.676  [task-1] INFO  com.easysoft.LockSupportDemo -do something...
```

（2）unPark\(\)操作在线程未启动时执行无效。

```
package com.easysoft;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.locks.LockSupport;

/**
 * TODO
 *
 * @author： zyp[2305658511@qq.com]
 * @date： 2019-11-08 12:53
 * @version： V1.0
 * @review: zyp[2305658511@qq.com]/2019-11-08 12:53
 */
@Slf4j
public class LockSupportDemo {


    public static class TaskThread extends Thread {
        public TaskThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            log.info("in thread[{}]  ", getName());
            log.info("park thread[{}]  ", getName());
            LockSupport.park(this);
            log.info("do something...");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        TaskThread t1 = new TaskThread("task-1");
        LockSupport.unpark(t1);
        t1.start();
        log.info("unpark thread[{}]  ", t1.getName());
        t1.join();
    }
}
```

输出结果：

```
2019-11-10 17:42:21.714  [main] INFO  com.easysoft.LockSupportDemo -unpark thread[task-1]  
2019-11-10 17:42:24.712  [task-1] INFO  com.easysoft.LockSupportDemo -in thread[task-1]  
2019-11-10 17:42:24.713  [task-1] INFO  com.easysoft.LockSupportDemo -park thread[task-1]  
```

（3）park\(\)不响应中断，发生线程中断，park\(\)操作不生效。

```
package com.easysoft;

import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.locks.LockSupport;

/**
 * TODO
 *
 * @author： zyp[2305658511@qq.com]
 * @date： 2019-11-08 12:53
 * @version： V1.0
 * @review: zyp[2305658511@qq.com]/2019-11-08 12:53
 */
@Slf4j
public class LockSupportDemo {


    public static class TaskThread extends Thread {
        public TaskThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            log.info("in thread[{}]  ", getName());
            log.info("park thread[{}]  ", getName());
            LockSupport.park(this);
            log.info("do something...");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        TaskThread t1 = new TaskThread("task-1");
        LockSupport.unpark(t1);
        t1.start();
        t1.interrupt();
        log.info("unpark thread[{}]  ", t1.getName());

        t1.join();
    }
}
```

运行结果：

```
2019-11-10 17:46:29.025  [main] INFO  com.easysoft.LockSupportDemo -unpark thread[task-1]  
2019-11-10 17:46:29.025  [task-1] INFO  com.easysoft.LockSupportDemo -in thread[task-1]  
2019-11-10 17:46:29.030  [task-1] INFO  com.easysoft.LockSupportDemo -park thread[task-1]  
2019-11-10 17:46:29.030  [task-1] INFO  com.easysoft.LockSupportDemo -do something...
```

（4）park\(\)与park\(Object blocker\)方法区别：设置一个线程和关联的blocker对象，blocker用来做分析，便于排查问题。![](/assets/locksupport-1.png)执行jstack pid；命令查看。

