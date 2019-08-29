[TOC]
# 说明
ThreadLocal无法解决多线程对某个共享变量访问的并发冲突问题，不能代替同步操作；它只是将一个共享变量在每个线程里copy一份，即每个线程都维护着变量的副本，每个线程都只访问自己的变量，空间换时间；
# 基本原理
1. 每个线程都维护了一个成员变量threadLocalMap，threadLocalMap本质上是一个map，key存储的当前threadLocal引用，value是缓存的数据。这样每个线程都只会从自己的threadLocalMap中获取数据，自然不会存在并发冲突问题；
2. threadLocalMap底层是一个Entry[]数组，Entry是一个持有ThreadLocal对象的WeakReference，即key是threadLocal对象，value是缓存的数据；这样当threadLocal不再被使用时，该对象能够被gc，避免内存泄漏；
3. Entry[]数组的长度是2的次幂，但是hash方法不同于hashmap，它是在AtomicInteger基础上累加固定长度的值0x61c88647作为hashcode，然后再&(len-1)得到数组对应的下标，之所以这样做，是为了进行完美的散列；
4. set方法时，首先判断map是否为null，是的话则初始化map和缓存数据，否则直接set数据；这里用的是懒加载，只有访问get/set时才会进行初始化操作(很合理，因为很多时候是不会用到threadLocalMap的)；根据hash算法得到数组下标，如果该位置为null，则直接赋值新的entry对象；否则比较两个entry对象的引用是否相同，相同则表明是同一个threadLocal，则覆盖value值；否则说明发生了hash冲突，这里采用的是线性探测再散列的方法解决hash冲突，((i + 1 < len) ? i + 1 : 0);即会寻找下个位置，如果到了数组末尾，则从数组头部开始；直至有坑位；注意：如果在查找过程中发现entry对象不为null，但是entry.get()为null，说明被gc了，则这时会清理脏数据value和对应数组位置置为null，size--等操作；
5. 添加完新的元素后，检测是否扩容，先进行脏数据清理，判断是否达到阈值(2/3的length)；清理完陈旧数据后，如果>=3/4阈值则进行扩容为原来长度的2倍，元素重新进行hash；
6. get操作时，首先从当前线程获取threadLocalMap，如果为null，则进行初始化方法；否则，获取值返回；
7. remove方法，从数组中删除是自身引用的key，最终是调用weakReference的clear方法，即将引用置为null，方便gc回收，然后再清除陈旧数据(清理key为null的数据，value和数组位置置为null)

# 内存泄漏
当一个线程调用ThreadLocal的set方法设置变量时候，当前线程的ThreadLocalMap里面就会存放一个记录，这个记录的key为ThreadLocal的引用，value则为设置的值。如果当前线程一直存在而没有调用ThreadLocal的remove方法，并且这时候其它地方还是有对ThreadLocal的引用，则当前线程的ThreadLocalMap变量里面会存在ThreadLocal变量的引用和value对象的引用是不会被释放的，这就会造成内存泄露的。但是考虑如果这个ThreadLocal变量没有了其他强依赖，而当前线程还存在的情况下，由于线程的ThreadLocalMap里面的key是弱依赖，则当前线程的ThreadLocalMap里面的ThreadLocal变量的弱引用会被在gc的时候回收，但是对应value还是会造成内存泄露，这时候ThreadLocalMap里面就会存在key为null但是value不为null的entry项。其实在ThreadLocal的set和get和remove方法里面有一些时机是会对这些key为null的entry进行清理的，但是这些清理不是必须发生的，因为是在set操作时会对key为null的数据进行清理，但是为了性能考虑，这里退出循环的条件是table中有null元素，也就是说，null元素后面的Entry里面key为null的元素不会被清理；
因此如果不再使用，最好手动调用remove方法；

# hash算法
原子类型的变量，每次增加一个固定的魔法值HASH_INCREMENT=1640531527作为hashcode采用，目的是为了能够让key均匀的分布在2的次幂的数组中。
斐波那契散列法，来保证哈希表的离散度。而它选用的乘数值即是2^32 * 黄金分割比（0.618）
数学推论以及大量实验证明。
一般来说做&，最高位都是0，避免hashcode是负数的影响。
```java
public static void main(String[] args) throws Exception {
    //黄金分割数 * 2的32次方 = 2654435769 - 这个是无符号32位整数的黄金分割数对应的那个值
	long c = (long) ((1L << 32) * (Math.sqrt(5) - 1) / 2);
	System.out.println(c);
    //强制转换为带符号为的32位整型，值为-1640531527
	int i = (int) c;
	System.out.println(i);
}
```

# 为什么推荐使用private static final来修饰ThreadLocal
ThreadLocal无法解决共享对象的更新问题，建议static修饰。这个变量是针对一个线程内所有操作共享的，所以设置为静态变量，所有此类的实例共享此静态变量，类在被加载时就会分配一块存储空间，所有此类的对象都可以操控这个变量。如果没有static，那么也不会存在冲突问题了。
ThreadLocal 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，也即变量在线程间隔离而在方法或类间共享的场景。
至于，是否使用 private 修饰与 ThreadLocal 本身无关。也就是说，是否使用 private 修饰是一个普遍的问题而不是与 ThreadLocal 有关的一个具体问题。

# 如何设计
”每个ThreadLocal类创建一个Map，然后用线程的ID作为Map的key，实例对象作为Map的value，这样就能达到各个线程的值隔离的效果“
JDK最早期的ThreadLocal就是这样设计的。（不确定是否是1.3）之后ThreadLocal的设计换了一种方式，也就是目前的方式，那有什么优势了：
1. 这样设计之后每个Map的Entry数量变小了：之前是Thread的数量，现在是ThreadLocal的数量，能提高性能，据说性能的提升不是一点两点(没有亲测)
2. 当Thread销毁之后对应的ThreadLocalMap也就随之销毁了，能减少内存使用量。