# Netty高性能原因
[Netty 系列之 Netty 高性能之道](https://www.infoq.cn/article/netty-high-performance)
1. Netty是异步非阻塞的，能够以较少的线程支持更多的并发连接数，且不会阻塞客户端请求；
2. 采用串行无锁化设计，即消息的处理尽可能在一个线程内完成，期间不会进行线程切换，避免了多线程的竞争和同步锁引起的性能下降；通过调整NIO线程池的参数，可以同时启动多个串行化的线程并行运行，这种局部无锁化的串行线程设计笔一个队列-多个工作线程模型性能更优。Netty的NioEventLoop读取到消息之后，直接调用ChannelPipeline的fireChannelRead(Object msg)，只要用户不主动切换线程，一直会由NioEventLoop调用到用户的Handler，期间不进行线程切换，这种串行化处理方式避免了多线程操作导致的锁的竞争，从性能角度看是最优的。
3. Netty是基于reactor模型设计的，采用事件驱动机制，selector类似一个观察者，当有事件发生时会通知其他线程处理；
4. 序列化性能的影响，java序列化方式相比其他性能会差很多，且通用性不好；Netty默认提供了对protobuf的支持，通过扩展Netty编码接口，用户可以实现其他的高性能序列化框架，如thrift的压缩二进制编解码框架。衡量序列化框架优劣的因素：序列化后的数据大小；序列化和反序列化的性能即cpu资源占用；是否支持跨语言；
5. 零拷贝方式，传统意义的零拷贝是不会在内核空间和用户空间进行数据拷贝；
    1. Netty 的接收和发送 ByteBuffer 采用 DIRECT BUFFERS，使用堆外直接内存进行 Socket 读写，不需要进行字节缓冲区的二次拷贝。如果使用传统的堆内存（HEAP BUFFERS）进行 Socket 读写，JVM 会将堆内存 Buffer 拷贝一份到直接内存中，然后才写入 Socket 中。相比于堆外直接内存，消息在发送过程中多了一次缓冲区的内存拷贝。

    2. Netty 提供了组合 Buffer 对象，可以聚合多个 ByteBuffer 对象，用户可以像操作一个 Buffer 那样方便的对组合 Buffer 进行操作，避免了传统通过内存拷贝的方式将几个小 Buffer 合并成一个大的 Buffer。

    3. Netty 的文件传输采用了 transferTo 方法，它可以直接将文件缓冲区的数据发送到目标 Channel，避免了传统通过循环 write 方式导致的内存拷贝问题。