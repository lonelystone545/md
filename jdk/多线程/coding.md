[TOC]
# 生产者消费者代码
[五种实现方式](https://juejin.im/entry/596343686fb9a06bbd6f888c)
# 实现阻塞队列
lock condition实现
```java
public class MyBlockingQueue {

    private Lock lock = new ReentrantLock();
    private Condition notFull = lock.newCondition();
    private Condition notEmpty = lock.newCondition();

    private Object[] datas = new Object[100];
    /**数组真实数据大小*/
    private int size;
    /**获取元素和存入元素两个指针*/
    private int takeptr, putptr;

    public void put(Object o) throws InterruptedException {
        try {
            lock.lock();
            //防止虚假唤醒，需要再次判断
            while (size == datas.length) {
                notFull.await();
            }
            datas[putptr++] = o;
            if(putptr == datas.length) {
                putptr = 0;
            }
            size++;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while(size == 0) {
                notEmpty.await();
            }
            Object res = datas[takeptr];
            datas[takeptr++] = null;
            if(takeptr == datas.length) {
                takeptr = 0;
            }
            size --;
            notFull.signal();
            return res;
        } finally {
            lock.unlock();
        }
    }
}
```