[TOC]
# 参考
[LockSupport源码解析](https://cloud.tencent.com/developer/article/1198491)
# 简介
LockSupport类，是JUC包中的一个工具类，用来创建锁或者其他同步类的基本线程阻塞原语。LockSupport类的核心方法其实就两个：park()和unark()，其中park()方法用来阻塞当前调用线程，unpark()方法用于唤醒指定线程。底层是依赖Unsafe实现。
需要在while循环中使用，重新检查条件是否满足，类似wait。
推荐采用park(blocker)的方式，此对象在线程受阻塞时被记录，以允许监视工具和诊断工具确定线程受阻塞的原因，这样方便从栈中看出阻塞的信息
# park/unpark/wait/notify/sleep区别
1. park和unpark是爱你原理是通过二元信号量做的阻塞(0,1)，所以只能最多只能是1。或者unpark时会释放许可证，park是获取许可证，unpark可以先于park调用，顺序无要求，不会发生线程无法唤醒的情况；而wait需要在notify前面，否则线程可能无法收到notify而处于一直等待状态；
2. LockSupport能够针对指定线程进行唤醒，而notify是随机唤醒一个线程，notifyAll是唤醒所有的线程。
3. LockSupport不需要在同步代码块里 。所以线程间也不需要维护一个共享的同步对象了，实现了线程间的解耦。
4. 都支持响应中断，但是park方法不会抛出异常。(也就是说如果当前调用线程被中断，则会立即返回但不会抛出中断异常)
这其实和Object类的wait()和signial()方法有些类似，但是LockSupport的这两种方法从语意上讲比Object类的方法更清晰，而且可以针对指定线程进行阻塞和唤醒。
# jdk对LockSupport的使用
## 锁AQS类中使用
## future.get
线程池执行任务，如果要获取结果通过future.get方法，如果任务还没结束，那么会阻塞，就是通过LockSupport.park()方法；任务执行完成后，通过cas操作将所有等待的线程拿出来，然后使用LockSupport.unpark()唤醒。
## 阻塞队列
如果线程池中没有任务时，那么线程都在干嘛呢？会调用队列的take方法阻塞等待新任务。以ArrayBlockingQueue为例，通过Lock的Condition.await()方法进行阻塞，但是其底层也是调用LockSupport.park()方法
# 使用
实现公平锁的简单版本

```java
public class FIFOMutex {
    private final AtomicBoolean locked = new AtomicBoolean(false);
    private final Queue<Thread> waiters = new ConcurrentLinkedQueue<Thread>();
 
    public void lock() {
        Thread current = Thread.currentThread();
        waiters.add(current);
 
        // 如果当前线程不在队首，或锁已被占用，则当前线程阻塞
        // NOTE：这个判断的意图其实就是：锁必须由队首元素拿到
        while (waiters.peek() != current || !locked.compareAndSet(false, true)) {
            LockSupport.park(this);
        }
        waiters.remove(); // 删除队首元素
    }
 
    public void unlock() {
        locked.set(false);
        LockSupport.unpark(waiters.peek());
    }
}
```