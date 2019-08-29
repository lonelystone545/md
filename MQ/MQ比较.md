# 选型
[选型需要考虑的要点](https://www.infoq.cn/article/kafka-vs-rabbitmq)
考虑的点：
1. 功能：延迟队列 死信队列 消息堆积和持久化 流量控制 消费模式是推还是拉
2. 性能：性能和功能上做trade-off。
    1. 吞吐量：rabbitMQ的单机QPS在万级别(5万左右)，而kafka在的单机QPS可以在十万级别，甚至百万。
    2. 时延：虽然一般使用MQ的场景不会要求时效性很好，但是如果时延较低，消费者能够快速消费消息，减少消息堆积，这样上下游的应用之间级联动作更加高效。
3. 可靠性：金融 支付领域，是无法容忍消息丢失的，kafka在极端情况下会丢消息，异步刷盘。日志处理、大数据场景常用kafka。
4. 可用性：无故障运行时间，几个9来衡量。kafka采用分区的多副本之间同步保证，rabbitMQ通过镜像队列。
5. 运维监控角度
6. 社区活跃度和成熟度
# Push & Pull
* Push
    优点：实时性好，服务端收到消息后立刻把消息推送给消费者，没有额外的延迟；
    缺点：
    1. consumer的消费速度是不一致的，由broker推送消息难以处理不同consumer的情况；

* Pull
    优点：
    1. consumer主动从broker上拉取消息，因此consumer能够根据自身的负载等状态决定从broker获取消息的频率；
    2. 能够聚合消息，减少网络开销；如果是push模式，broker在收到消息后会离开推送给消费者，无法对消息聚合后再推送给消费者，而pull模式，由于是消费者主动获取消息，所以一次可以拉取多条消息；
    缺点：实时性不好，有额外的延迟。如果提高pull频率，可能在没有消息的时候产生大量pull请求；如果降低pull频率，那么自然有延迟了。
    
* Long-Polling 长轮询
是pull模式的变种，解决实时性问题。
consumer主动发起请求到broker，正常情况下，broker返回消息给consumer，然后立刻发起下一次pull请求；但是在没有消息时，能够将请求阻塞在broker，在产生下一条消息或者超时之前，响应给consumer；
这个避免了多余的pull请求，也解决了实时性问题。

* 优化
Long-Polling情况下，消息到达broker后，最坏情况下是经过3T(T为单次网络传输时间)的延迟才能到消费者。(消息到broker后，pull请求刚刚返回给consumer，然后consumer发起pull请求，消息才又返回给consumer)
改进：consumer维护一个buffer，pull从broker获取的数据要写到buffer，consumer消除从buffer中获取消息进程处理。这样每次pull请求时可以把buffer的剩余空间告诉给broker，这样broker中有消息后，能够根据buffer剩余空间大小主动推送消息给consumer
    
    
# RocketMq
java开发，具有高吞吐量、高可用性，适合大规模分布式系统应用的特点。在阿里内部广泛用于交易 充值 流计算 消息推送 日志处理 binlog分发等场景。