# HashSet
1. 底层采用HashMap实现，map的key就是set集合中的元素，map的value是一个全局静态的常量private static final Object o = new Object()，是一个虚拟的对象，空对象在堆中占用8字节； <font color="#ff00">为什么不用null作为value呢？</font>

    ```java
        public boolean remove(Object o) {
            return map.remove(o)==PRESENT;
        }
    ```
如果value填充null了，那么map在remove一个不存在的key时也会返回null，这样就得要求set集合在删除数据之前先判断key是否存在了，增加了额外的开销。

1. 不保证元素的顺序，允许使用null元素；
2. 迭代器iterator是fail-fast机制；

# TreeSet
1. 基于TreeMap实现，时间复杂度O(logN)，底层红黑树，是一种二叉排序树
2. 非线程安全，支持浅拷贝，序列化
3. fail-fast机制
4. 是不可重复集合，默认是按照字典顺序进行排序，也可以按照指定的排序方式
5. value也是用空对象表示
6. add方法调用treeMap.put方法，首先会按照二叉搜索树的方式插入新的Entry对象，然后再按照红黑树的颜色进行调整树的结构；

    ```java
    do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
    ```
7. TreeSet集合中元素需要实现Comparable接口或者自定义Comparator比较器，这样就不需要元素去实现比较接口了
