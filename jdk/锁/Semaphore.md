[TOC]
# 参考
[https://segmentfault.com/a/1190000015918459](https://segmentfault.com/a/1190000015918459)
# 总结
1. AQS的state对于Semaphore来说就是permits许可证，采用了共享锁实现。
2. Semaphore是一个有效的流量控制工具，它基于AQS共享锁实现。我们常常用它来控制对有限资源的访问。每次使用资源前，先申请一个信号量，如果资源数不够，就会阻塞等待；每次释放资源后，就释放一个信号量。