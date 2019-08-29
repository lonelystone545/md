[TOC]
# 简介
ReentrantReadWriteLock类，顾名思义，是一种读写锁，它是ReadWriteLock接口的直接实现，该类在内部实现了具体独占锁特点的写锁，以及具有共享锁特点的读锁，和ReentrantLock一样，ReentrantReadWriteLock类也是通过定义内部类实现AQS框架的API来实现独占/共享的功能。
# 特性
1. 支持公平/非公平锁；
2. 支持锁的重入：同一读线程在获取了读锁后还可以获取读锁，同一写线程在获取了写锁之后既可以再次获取写锁又可以获取读锁；
3. 支持锁降级：先获取写锁，然后获取读锁，最后释放写锁，这样写锁就降级成了读锁。但是，读锁不能升级到写锁；即写锁能够降级为读锁，读锁不能升级为写锁。因为读锁可能会被其他线程获取，此时无法加写锁。[锁降级详解，有case](https://www.jianshu.com/p/0f4a1995f57d)

```java
    class CachedData {
   Object data;
   volatile boolean cacheValid;
   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock();
     if (!cacheValid) {
        // Must release read lock before acquiring write lock
        rwl.readLock().unlock();
        rwl.writeLock().lock();
        try {
          // Recheck state because another thread might have
          // acquired write lock and changed state before we did.
          if (!cacheValid) {
            data = ...
            cacheValid = true;
          }
          // Downgrade by acquiring read lock before releasing write lock
          rwl.readLock().lock();
        } finally {
          rwl.writeLock().unlock(); // Unlock write, still hold read
        }
     }

     try {
       use(data);
     } finally {
       rwl.readLock().unlock();
     }
   }
 }
```
如果在加了写锁后没有加读锁，直接释放了写锁，此时如果其他线程也处理了数据data，那么当前线程是无法感知的，为了保证数据可见性，所以需要加读锁。或者，use(data)部分也放在写锁中执行完成后再释放，但是其他线程就无法读了，效率低。
可以用读写锁封装TreeMap为线程安全的集合，提高并发访问。

# 源码
[参考-主要](https://juejin.im/post/5b7d659c6fb9a019fc76dfba)
[多线程读写锁案例讲解--有问题](https://segmentfault.com/a/1190000015807600)
## 基本属性
```java
    /**state-高16位表示读锁的个数，低16位表示写锁的个数，读锁和写锁的最大数量为2^16-1*/
    private volatile int state;
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
```
可以看出，ReentrantReadWriteLock对state的定义是读写锁状态，高16位表示读锁个数，低16位表示写锁个数，这里个数将重入锁也计算在其中了。当然也可以通过threadLocal获取到每个线程的重入次数。

## ReadLock
### lock()
* 调用AQS的tryAcquireShared()方法判断能否获得读锁，条件如下：
    1. 判断写锁是否被其他线程占用（可以被本线程占用，属于写锁降级

        ```java
            exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current
        ```
    2. 判断读锁是否需要阻塞（readerShouldBlock，公平和非公平），读锁数量小于最大值，cas更新读锁数量
         ```java
        !readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)
         ```
         readerShouldBlock()方法在公平锁时，判断等待队列中是否有其他线程正在排队，如果有则应该阻塞当前线程(此时肯定有写锁获取了锁，队列中等待的可能是读锁也有可能是写锁)；在非公平锁时，判断等待队列中的第一个后继结点是否是获取写锁的，如果是，则优先让写锁来（此时可能是读锁被获取，或者写锁被获取了），避免写锁饥饿，如果不是写锁，则随便抢，因为是非公平模式。
    3. 满足上面两个条件后，然后更新其他属性值，如每个线程都维护了获取读锁的次数(threadLocal实现)等；
    4. 如果第二个条件中进行cas操作失败，说明读锁并发或者读写冲突，那么会调用fullTryAcquireShared方法，该方法是对tryAcquireShared方法的补充，通过自旋重试CAS操作。退出条件：期间锁被其他写线程获取了，或者读锁获取成功
* 如果读锁获取失败则调用doAcquireShared方法进行队列中排队
类似CountDownLatch的共享锁进行排队，将当前线程封装为共享模式的Node结点插入队列尾部，如果被唤醒了，则会判断自身是否为头节点的后继结点，是，则尝试再次获取读锁，获取成功后会依次唤醒其后继结点的其他共享模式的Node结点，每个共享模式结点都会这样进行唤醒。但是如果其后继结点是个写锁，那么没办法继续向下唤醒了，即使后面还有共享结点。

### unlock()
* 调用AQS的tryReleaseShared方法释放读锁。这里做了一些优化操作，避免每次从threadLocal中获取线程id和读锁次数的关系(firstReader&firstReaderHoldCount || cachedHoldCounter)，然后cas更新state值，更新完成后，如果state==0表明读锁和写锁都被释放了，则执行doReleaseShared操作；否则不会进行唤醒的操作。
* 掉头AQS的doReleaseShared进行唤醒，检查head结点的waitStatus是否为Signal，然后唤醒其后继结点。

## WriteLock
### lock()
1. 判断是否能够获得写锁，如果有读锁占用或者其他线程占用了写锁，则该线程获取写锁失败；如果写线程数量超过了最大值则抛异常；CAS更新state变量，更新成功则表明获得了写锁；判断写锁是否需要阻塞(对于重入线程来说不需要判断)，若是公平锁，则判断等待队列中是否有其他线程等待；若是非公平锁，则直接放行；写线程优先级高；最后设置当前线程为锁的占有者setExclusiveOwnerThread，读锁/共享锁不会设置，因为有很多线程哦～
2. 如果获取写锁失败，则以独占锁的方式进入队列等待（可以参考ReentrantLock的实现）

### unLock()
比较简单，将state减1，然后判断独占锁的数量是否为0，为0表示重入锁释放完毕则会唤醒队列首个线程；注意这里是是判断独占锁数量而不是判断state，因为存在锁降级，即线程获得了写锁后获得了读锁，这个是允许的，此时state！=0；
这里如果队列的首个线程是读锁，那么读锁线程重新获得锁后，会继续释放后继结点共享模式的线程；

# 问题
    1. 写锁饥饿问题
        这里不会发生写锁饥饿问题，写锁的优先级比读锁高；即线程在获取读锁时，不论是公平还是非公平，对于等待队列中已经有了写锁的线程，是无法获得读锁的，只能排队等待；读多写少场景，防止写线程饿死一直无法获得锁。
    2. 如何支持锁降级
        同一个线程允许获得写锁后再获得读锁，不会发生死锁。因为在获取读锁时，对于自身线程是允许在拥有写锁的前提下获得读锁的；但是不支持升级，即先获得读锁再获得写锁，因为可能会有很多线程拥有这个读锁，而不仅仅是当前这个线程。
    3. 死锁问题
    4. 为什么用一个变量state表示资源状态呢
        1. 内部类Sync继承了AQS，在AQS中只有state表示同步器的状态，其子类会对这个资源状态有不同的解释；
        2. 如果两个变量表示，那么在进行CAS操作时无法保证对两个变量cas的原子性，对一个变量进行CAS更为方便；两个变量时，需要保证对其中一个cas后，另外一个变量期间没有发生改变，如果发生了改变还得需要进行回滚；
    5. 线程A获得读锁后，线程B一定能够拿到读锁么？
        不一定。如果线程A获得读锁后，线程C尝试获得写锁失败，则排队等待；若采用公平锁，则线程B获得读锁会失败进入队列等待；若采用非公平锁，由于线程C是等待队列的首个结点且是写线程，那么线程B获得读锁也会失败。这样防止了写线程饿死的情况，因为读写锁适应的场景是读多写少，如果没有这个机制，那么读锁会一直拿到锁不被释放，写线程一直没办法进行。

