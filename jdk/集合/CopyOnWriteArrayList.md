[TOC]

[参考------](https://segmentfault.com/a/1190000016214572)
# 并发list
1. 使用Vector类
2. 使用Collections.synchronizedList返回list集合，装饰者模式
3. 自己实现ArrayList子类，同步。加锁

# CopyOnWriteArrayList简介
## 介绍
1和2都是加全局锁，读写/写写/读读互斥，并发性能较低；
适应于 读多写少 的业务场景，采用 写时复制 的思想，在需要修改/增加/删除元素时，先copy一份原列表，然后在副本上修改，修改完成后，再将引用从原列表指向新列表；
读写不会冲突，可以并发进行，读操作还是在原列表，写操作在新列表，仅仅当有多个线程同时进行写的时候，才会进行同步；

## 优点
读没有加锁，性能非常好；写的时候才会加锁，读写不互斥；通过 写时复制 的思路，延迟更新，保证数据的最终的一致性，也就是说，在性能和一致性上做了trade-off。
之所以没有采用 读写锁，因为如果有写线程在进行，那么后续的读写都会被阻塞；

## 弊端
1. 内存占用，使用了 写时复制 的思想，所以在进行写操作时，内存中会同时存在两个array数组，可能会造成频繁gc，所以不适合大数据量的场景；
    1. 数据一致性，只能保证最终一致性，不能保证实时一致性-读操作读的是旧数组的数据，如果希望写入的数据能够立刻被读到，那么copyOnArrayList并不适合。(Unsafe类getObjectVolatile应该可以解决)

## COW VS 读写锁
相同点：1. 都是通过读写分离的思想实现 2. 读线程之间互不阻塞
不同点：读写锁，为了数据的实时性考虑，在写锁获取后，读线程会等待，或者，读锁被获取后，写线程会等待，从而避免 脏读 问题，即读写锁只能做到读读不互斥，无法做到读写不互斥；而COW牺牲了数据实时性，保证最终一致性，读线程不会存在等待情况；

## 为什么复制一个新的数组
1. 如果只用一个数组肯定能够实现线程安全的list，但是可能会比较复杂，这样实现起来比较简单，根据业务场景自行选择；
2. “快照”风格的迭代器方法在创建迭代器时使用了对数组状态的引用。此数组在迭代器的生存期内不会更改，因此不可能发生冲突，并且迭代器保证不会抛出ConcurrentModificationException。创建迭代器以后，迭代器就不会反映列表的添加、移除或者更改。在迭代器上进行的元素更改操作（remove、set和add）不受支持。这些方法将抛出UnsupportedOperationException。

# 源码
```java
    /** 排他锁，用于同步修改操作*/
    final transient ReentrantLock lock = new ReentrantLock();

    /** 内部数组，原子变量，对数组引用具有可见性 */
    private transient volatile Object[] array;
```
## get
直接返回array[index]，没有加锁；
## add
```java
public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
写时先加锁，保证只能有一个线程进行修改，然后创建新的数组(len+1)，并将原数组的值复制到新数组，新元素添加到新数组最后，修改array引用指向新数组。
## remove
删除方法和插入一样，都需要先加锁（所有涉及修改元素的方法都需要先加锁，写-写不能并发），然后构建新数组，复制旧数组元素至新数组，最后将array指向新数组。
## iterator迭代
CopyOnWriteArrayList对元素进行迭代时，仅仅返回一个当前内部数组的快照，也就是说，如果此时有其它线程正在修改元素，并不会在迭代中反映出来，因为修改都是在新数组中进行的。所以，迭代过程中不会抛出并发修改异常——ConcurrentModificationException



