[TOC]
# 调优技巧
* 避免使用 %C-输出调用者的全限定名 %L-输出行号 %m-输出方法名，因为这些信息是无法直接拿到的，logback必须通过一些特殊的手段去获取这些数据(如在日志打印出产生一个堆栈信息)，这样会对性能有一定的损耗
* 移除控制台输出 <appender-ref ref="STDOUT"/> ==> 官方测试是不同数量级
* 调整日志输出格式
   a) logger.debug("name = {}", name);
   b) logger.debug("name = " + name);
   a方式只会在该语句生效时才会进行字符串拼接，而b方式始终都会进行字符串拼接。
* 动态配置，仅允许开发环境输出到控制台
* 从源码来看，异步写日志时，消费者每次从队列中获取一个loggingEvent，可以一次获取多个然后写入日志文件中，减少磁盘IO;(每次取多少数量不好控制，无法知道队列的大小)

```xml
<root level="DEBUG">
    <appender-ref ref="ASYNC"/>
    <if condition='property("os.name").toUpperCase().contains("WINDOWS") || property("os.name").toUpperCase().contains("MAC")'>
        <then>
            <appender-ref ref="STDOUT"/>
        </then>
    </if>
</root>
```
添加依赖

```xml
<dependency>
    <groupId>org.codehaus.janino</groupId>
    <artifactId>janino</artifactId>
    <version>3.0.11</version>
</dependency>
```
* 异步写日志
   发现的原因在于：云硬盘抖动，接口出现大量超时，排查代码和配置，发现目前系统中采用的是同步写日志。而且，对于同步写日志，每写一次就会发生一次磁盘IO，性能损耗比较大。采用异步方式，不会阻塞业务线程，提高性能。
   
# 异步写日志
## 配置
更新logback版本：<logback.version>1.2.3</logback.version>
```xml
<appender name="asyncCommonAppender" class="ch.qos.logback.classic.AsyncAppender">
        <!--丢弃的阈值,若队列剩余长度小于该值，则丢弃<=info级别的日志,保留warn/error级别日志-->
        <discardingThreshold>100</discardingThreshold>
        <!--阻塞队列长度-->
        <queueSize>1024</queueSize>
        <!--获取调用者的信息,行号(需要从栈中获取)等,为了提升性能，设置false，当logevent进入队列后，event关联的调用者数据不会被提取，只有线程名这些简单的数据-->
        <includeCallerData>false</includeCallerData>˝
        <!--只能一个,多个无效,以第一个为准。保证消费者只有一个-->
        <appender-ref ref="CONSOLE"/>
        <!--queue.offer非阻塞入队，可能会丢失所有级别日志-->
        <neverBlock>false</neverBlock>
</appender>
```
## 基本原理
当有LoggingEvent进入AsyncAppender后，该appender调用append方法，该方法中首先判断当前buffer的容量和日志级别是否允许丢弃，这样当消费能力不足导致buffer快满的时候，能够将debug/info级别日志丢弃；否则，则会将LoggingEvent放入buffer中。（buffer采用ArrayBlocking实现）===> 可见，AsyncAppender只是负责将log事件放入buffer中，起生产者的作用；并不处理日志；
系统在启动时，AsyncAppender会在内部创建一个Worker线程，该线程充当消费者的作用，从buffer中获取LoggingEvent，然后其他的appender负责处理日志事件。
## 问题
1. 异步写日志时，消费者每次从队列中获取一个loggingEvent，可以一次获取多个然后写入日志文件中，减少磁盘IO;(每次取多少数量不好控制，无法知道队列的大小，但是既然是异步的，其实也无所谓，可以设置每次获取的数量以及超时时间，增加复杂度）
2. 如果系统断电或者啥，则内存队列会丢失；

## 其他做法
多线程日志，按线程（或线程取模）记录日志，减少竞争，日志也能保证持久化
[多线程logback调优](https://segmentfault.com/a/1190000016204970)

## 为什么采用ArrayBlockingQueue
logback作为日志框架，性能和稳定性是首选的。ArrayBlockingQueue虽然是一把锁，但是在高并发请求时，不会进行额外的垃圾回收，对gc影响小；数组分配的是连续内存空间，访问的效率高；入队和出队的操作比较简单，不会有额外的Node对象；
测试来看，LinkedBlockingQueue的大部分时优于ArrayBlockingQueue，但是不够稳定；
# 源码


   