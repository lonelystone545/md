# 参考
[jdk8的linkedList实现](http://www.tianxiaobo.com/2018/01/31/LinkedList-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-JDK-1-8/)
# 基本知识点
1. 实现了List/Deque/Cloneable/Serializable接口，能够将LinkedList作为双端队列使用，底层实现是双向链表；浅拷贝；
2. 存储时采用Node节点，val/next/pre指针；
3. 迭代器模式：提供一种方法可以集合对象中的各个元素，而不用暴露这个对象的内部表示；LinkedList的迭代器有三个：Iterator、ListIterator、DescendingIterator；其中DescendingIterator与Iterator只能单向遍历，遍历的方向相反。而ListIterator可以双向遍历。
```java
private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;
        private Node<E> next;
        private int nextIndex;
        private int expectedModCount = modCount;

        ListItr(int index) {
            // assert isPositionIndex(index);
            next = (index == size) ? null : node(index);
            nextIndex = index;
        }

        public boolean hasNext() {
            return nextIndex < size;
        }

        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }

        public boolean hasPrevious() {
            return nextIndex > 0;
        }

        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();

            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }
```