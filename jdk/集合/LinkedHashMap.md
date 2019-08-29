# 基本原理
LinkedHashMap继承了HashMap，在HashMap基础上维护了一条双向链表(非循环)，解决了HashMap不能随时保持遍历顺序和插入顺序不一致的问题。同时，它也对访问顺序提供支持，在缓存中有用。
# 主要参考
[jdk8 linkedHashMap源码解析](https://segmentfault.com/a/1190000012964859)
# 总结
1. linkedHashMap在原有的hashmap结构(数组+链表/红黑树)基础上，在每个entry对象中增加了before和after指针，可以用来访问前驱和后继节点；
2. entry对象继承结构:HashMap.TreeNode--->LinkedHashMap.Entry--->HashMap.Node--->Map.Entry
3. 链表建立过程：开始时，LinkedHashMap的head和tail同时指向新节点，随着节点的不断插入，新节点链接在tail引用的后面，移动tail到尾部，可以实现链表的更新。
4. put方法，调用父类HashMap方法，但是hashMap中的Node没有before和after指针，因此LinkedHashMap重写了父类的newNode()方法，每次插入一个节点时，都会插入到双向链表的尾部；
5. HashMap在进行增删查的时候预留了回调接口，允许在这些操作后执行指定的操作;

    ```java
        // Callbacks to allow LinkedHashMap post-actions
        void afterNodeAccess(Node<K,V> p) { }
        void afterNodeInsertion(boolean evict) { }
        void afterNodeRemoval(Node<K,V> p) { }
    ```
6. 删除调用remove方法时，依然是调用父类Hashmap.remove实现，但是父类中并不会删双向链表，LinkedHashMap重写了afterNodeRemoval方法，在该方法中删除双向链表：如果该节点为头节点，则下个节点作为头；如果是尾部节点，则上个节点作为尾；如果是中间某个节点，则上个节点和下个节点互相关联，最后将自身节点前驱和后继置为null。
7. 查询get时，调用HashMap.getNode方法实现，但是LinkedList支持按照访问顺序进行排序，在初始化时指定accessOrder为true。只需要在访问时，将该节点移动到链表的尾部即可。afterNodeAccess
8. afterNodeInsertion方法是在插入之后进行回调的，如果作为缓存，可以设置返回true，那么会从双向链表头部将节点移除。


