# ArrayList为什么实现RandomAccess接口
该接口里面没有实现方法，只是起一个表识的作用，可以用注解代替，实现该接口的类表明可以进行快速的随机访问。
比如Collections.binarySearch()方法，对于实现RandomAccess的集合会采用下标读取数据；否则采用迭代器iterator读取数据（迭代器方法：需要移动指针，但是并不是每次从头活着尾移动，而是从当前位置向前或者向后移动）
ArrayList实现了RandomAccess接口，底层是基于数组，查询O(1)，而LinkedList底层是基于双向链表访问，查询O(n)；所以对于LinkedList来说，迭代器的访问方式比下标更快；因为LinkedList不会真正的维护一个下标，只是根据传递进来的下标和链表长度，决定是从链表头部还是尾部进行遍历访问，也就是说，如果遍历多次，那么每次都要从头开始访问；而迭代器不同，迭代器则顺着链表的指针节点，一个个进行遍历，不需要每次从头开始访问。

# 向指定位置添加元素
在头部加入数据时，linkedlist较快，遍历插入位置的花费时间小，而arrayList需要将元数组后面的所有元素拷贝一次；
在尾部插入数据时，数据量比较小时，linkedList比较快，直接从尾部插入数据即可，而arrayList需要频繁扩容；当数据量大时，arraylist比较快，因为扩容后是1.5倍，大容量扩容后一次能够提供很多空间；也就是说，当arraylist不需要扩容时，效率比linkedlist高，因为直接是数组元素赋值，不需要new Node；
插入位置靠近中间时，linkedlist效率比较低，因为遍历时是从头或者尾部开始，arraylist效率可能比linkedlist高；

LinkedList在向指定位置添加元素时，首先会根据插入的位置判断是从前还是从后遍历（index 与 size>>1 的大小，决定是从头部还是尾部开始遍历)

# System.arrayCopy
是native方法，jvm原生的方法，效率很高，连续内存拷贝，直接赋值；