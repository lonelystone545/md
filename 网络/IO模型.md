# 参考
[聊聊Linux 五种IO模型](https://www.jianshu.com/p/486b0965c296)
[Java NIO浅析](https://tech.meituan.com/2016/11/04/nio.html)
NIO底层就是IO多路复用，而IO多路复用也是依赖select/poll/epoll实现的，是一种同步非阻塞IO；

# tomcat
tomcat的调优：BIO NIO AIO
NIO使用的就是IO多路复用，允许通过少量的线程处理大量的连接请求。但是如果连接数比较少，NIO的性能不一定会比BIO好，因为NIO需要两次系统调用，而BIO只需要一次；连接数大的情况下，肯定是NIO好，能够支持更多的连接。BIO的连接数和线程池的个数是一致的，一个请求分配一个线程进行处理，当并发树较大时，需要创建大量的线程来处理连接，系统资源占用较大，且如果当前线程暂时没有数据可读，那么线程会阻塞在read操作上，造成线程资源浪费。

# select/poll/epoll
1. select打开的fd有数量限制，32位系统默认是1024，64位系统默认是2048；
2. select和poll需要轮询所有的fd集合，找到已经就绪的fd，效率比较低；
3. epoll没有2048这个限制，它所支持的fd上限是最大可以打开文件的数目，与机器的内存有关；
4. epoll会通过回调函数，当某个fd就绪后，通过回调函数通知该fd即可，效率高；
5. epoll使用mmap加速内核与用户空间的消息传递，select、poll和epoll都需要内核把fd消息通知给用户空间，epoll通过内核在用户空间mmap同一块内存实现的。

# netty底层实现
异步非阻塞

一个连接的完整网络处理过程包括：accept接受连接、read读取数据、decode解码、process处理数据、encode编码、send发送数据这几步，reactor将这些步骤拆分成小的task，采用非阻塞方式执行。
reactor采用事件驱动机制，当某个事件准备就绪后，reactor收到对应的网络事件通知，并将task分发给绑定了对应网络事件的handler执行。
reactor：负责响应事件，将事件分发给绑定该事件的handler处理；
handler：事件处理器，绑定了某类事件，负责执行对应事件处理；
acceptor：handler的一种，绑定了connect事件，当客户端发起connect请求时，reactor将accept事件分发给acceptor处理；

[netty学习系列二：NIO Reactor模型 & Netty线程模型](https://www.jianshu.com/p/38b56531565d)
[Netty Reactor模型](https://www.jianshu.com/p/87f438abbd5d)
[这可能是目前最透彻的Netty原理架构解析](https://juejin.im/post/5be00763e51d453d4a5cf289)
[Netty 实现原理浅析](https://cloud.tencent.com/developer/article/1031640)
[reactor模型](https://essviv.github.io/2017/01/25/IO/netty/reactor%E6%A8%A1%E5%9E%8B/)