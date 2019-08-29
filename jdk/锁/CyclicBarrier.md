[参考1](https://segmentfault.com/a/1190000015888316)
[参考2](https://segmentfault.com/a/1190000016518256)
# 总结
1. CyclicBarrier是一个栅栏，让线程到达栅栏时被阻塞(await)，直至到达栅栏的线程数满足指定数量要求时，栅栏才会打开放行。用于线程之间的调度。
2. 只要在栅栏上等待的任何一个线程抛出了异常，那么Barrier会认为凑不齐所有的线程，就会将栅栏置为损坏Broken状态，并将BrokenBarrierException传递给其他所有正在等待await的线程。因此在使用CyclicBarrier时，需要对异常进行处理，有适当的重试机制。
3. 底层采用ReentrantLock和Condition实现，独占锁的方式。

# CyclicBarrier & CountDownLatch
1. CountDownLatch是一个计数器，初始化一个初始值，调用await方法的线程会被阻塞，调用countDown方法会将计数器减一，如果减至0了，那么所有阻塞的线程会被唤醒，向下执行；CyclicBarrier是栅栏，也会初始化一个初始值，调用await方法的线程会在栅栏处阻塞，如果最后一个线程到达了栅栏，那么所有线程都会被唤醒，继续执行；CountDownLatch是让所有线程等待某一个条件完成时然后一起向下运行，而CyclicBarrier是线程之间的协调；
2. CountDownLatch底层采用AQS的共享锁实现，CyclicBarrier采用ReentrantLock和Condition实现，即独占锁和条件队列实现。CountDownLatch的线程被唤醒是共享锁的唤醒，唤醒后可以立刻执行，不需要等待锁释放，而CyclicBarrier的线程在条件队列被唤醒后，需要先获取锁，从条件队列移动到等待队列，等待前驱结点释放锁后，后继结点才能获取到锁。
3. CountDownLatch是一次性的，当count值减少为0时，不会被重置；而CyclicBarrier在所有线程通过栅栏时，会开启新的一代，count值会被重置，可以重复使用。