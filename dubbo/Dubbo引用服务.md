[TOC]
# 参考
[Dubbo中消费者初始化的过程解析](https://cxis.me/2017/03/21/Dubbo%E4%B8%AD%E6%B6%88%E8%B4%B9%E8%80%85%E5%88%9D%E5%A7%8B%E5%8C%96%E7%9A%84%E8%BF%87%E7%A8%8B%E8%A7%A3%E6%9E%90/)--这个主要是源码解析
[dubbo剖析：二 服务引用](https://www.jianshu.com/p/a4f1a504fa95)--这个简单明了
# 总结
初始化过程就是，消费者会注册到注册中心，订阅服务，连接服务提供者端，创建代理对象，代理对象由spring管理。为后面就是服务消费者和提供者之间的通信做准备。

Dubbo 服务引用的时机有两个，第一个是在 Spring 容器调用 ReferenceBean 的 afterPropertiesSet 方法时引用服务，第二个是在 ReferenceBean 对应的服务被注入到其他类中时引用。这两个引用服务的时机区别在于，第一个是饿汉式的，第二个是懒汉式的。默认情况下，Dubbo 使用懒汉式引用服务。如果需要使用饿汉式，可通过配置 <dubbo:reference> 的 init 属性开启。下面我们按照 Dubbo 默认配置进行分析，整个分析过程从 ReferenceBean 的 getObject 方法开始。当我们的服务被注入到其他类中时，Spring 会第一时间调用 getObject 方法，并由该方法执行服务引用逻辑。按照惯例，在进行具体工作之前，需先进行配置检查与收集工作。接着根据收集到的信息决定服务用的方式，有三种，第一种是引用本地 (JVM) 服务，第二是通过直连方式引用远程服务，第三是通过注册中心引用远程服务。不管是哪种引用方式，最后都会得到一个 Invoker 实例。如果有多个注册中心，多个服务提供者，这个时候会得到一组 Invoker 实例，此时需要通过集群管理类 Cluster 将多个 Invoker 合并成一个实例。合并后的 Invoker 实例已经具备调用本地或远程服务的能力了，但并不能将此实例暴露给用户使用，这会对用户业务代码造成侵入。**此时框架还需要通过代理工厂类 (ProxyFactory) 为服务接口生成代理类，并让代理类去调用 Invoker 逻辑(invoke方法)。避免了 Dubbo 框架代码对业务代码的侵入，同时也让框架更容易使用**。

# 服务引用源码解析
## 入口
当spring遇见自己无法解析的标签时，会调用parseCustomElement解析dubbo标签，最终是交给DubboNamespaceHandler类(实现了NamespaceHandlerSupport接口)进行解析dubbo标签，在DubboNamespaceHandler类中，对不同的标签采用不同的Bean进行解析，DubboBeanDefinitionParser实现了BeanDefinitionParser接口，重写parse方法返回BeanDefinition交给spring管理。
解析reference标签的是：
```java
new DubboBeanDefinitionParser(ReferenceBean.class, false)
```

## ReferenceBean
该类实现了ApplicationContextAware接口，spring会在bean实例化时调用setApplicationContext方法，将applicationContext交给SpringExtensionFactory管理，最终当需要通过dubbo spi获取bean时就是调用SpringExtensionFactory的applicationContext来获取。
`ExtensionFactory有三个实现类SpringExtensionFactory和SpiExtensionFactory-java spi机制，通过ExtensionLoader加载类，AdaptiveExtensionFactory被标记了@Adaptive注解，则会默认加载该实现类，该类管理了上面的两个类，以后获取扩展点时都通过该类即可，相当于门面模式`
ReferenceBean还实现了InitializingBean接口，bean初始化完成后会回调afterPropertySet方法；还实现了FactoryBean接口，可以在后期获取bean时做一些操作，dubbo在这个时候进行初始化。还实现了DisposableBean接口，bean销毁时调用destroy方法。

`FactoryBean是一个工厂Bean，和普通bean不同，FactoryBean返回的是getObject方法返回的bean，即工厂bean是可以生产bean的工厂。而BeanFactory是bean的工厂，管理了spring的bean。
FactoryBean一般用来创建逻辑比较复杂的bean，一般的bean直接用xml配置即可，但是如果一个bean的创建过程涉及到很多其他的bean和复杂的逻辑，用xml配置比较困难，可以考虑用FactoryBean。`

### Dubbo消费者初始化时机
1. reference标签没有配置init属性，默认为null，此时是延迟初始化的，即等到该bean引用被注入到其他bean中或者调用getBean方法获取这个bean时(会调用FactoryBean.getObject()方法)，才会触发服务引用的初始化。
2. reference标签的init属性为true，则bean在完成初始化后会立即进行dubbo服务引用的初始化操作。

### 初始化流程
调用ReferenceConfig对该服务引用进行初始化操作。
首先检查初始化所有的配置信息，然后调用ref=createProxy()方法创建代理，消费者最终得到的就是服务的代理，初始化做的事情主要就是引用对应的远程服务，大概步骤如下：
1. 监听注册中心
2. 连接服务提供者端进行服务引用
3. 创建服务代理并返回

`zk作为注册中心时，消费者启动时做的事情：
订阅/dubbo/com.foo.BarService/providers目录下的提供者URL地址。
向/dubbo/com.foo.BarService/consumers目录下写入自己的URL地址。`

* createProxy
createProxy()方法中首先判断是否是本地服务引用(dubbo reference配置了injvm参数，scope参数)，如果是本地服务引用则拼接URL的protocol为injvm,localhost,端口为0，调用Protocol的refer方法获得invoker对象，赋值给dubbo reference的ref，即消费者调用的是invoker。
然后判断是否是点对点直连(dubbo reference url或者jvm启动参数-D映射服务地址或者配置properties文件中包含多个服务并使用-Ddubbo.resolve.file指定文件位置)；
```java
<dubbo:reference id="xxxService" interface="com.alibaba.xxx.XxxService" url="dubbo://localhost:20890" />
java -Dcom.alibaba.xxx.XxxService=dubbo://localhost:20890
xx.properties:com.alibaba.xxx.XxxService=dubbo://localhost:20890
```
如果是非点对点直连，则获取所有的注册中心，并将所有注册中心的url加到list集合中；
如果只有一个url则直接调用远程服务refProtocol.refer方法(注意：如果是点对点直连，则这里的url就是所配置的协议如dubbo，否则，就是注册中心的url，协议为registry，需要去注册中心订阅服务)，否则需要对每个注册中心都要执行这个方法，cluster模式(后面讲解)；

最后返回服务代理proxyFactory.getProxy(invoker)；

* refProtocol.refer方法
该方法首先进入ProtocolListenerWrapper的refer方法，然后在进入ProtocolFilterWrapper的refer方法，然后再进入RegistryProtocol的refer方法，这里的url协议是registry，所以上面两个Wrapper中不做处理，直接进入了RegistryProtocol。
由于是采用zk作为注册中心，这里会把url的协议替换为zk的url即registry替换为zookeeper，然后通过url连接注册中心(有本地缓存，只有第一次初始化时才会连接然后放进缓存，后续则直接返回。key为协议+ip+端口)，将消费者注册至注册中心。
默认使用zkClient，ZookeeperRegistry继承了FailbackRegistry，FailbackRegistry继承了AbstractRegistry；

`ZookeeperRegistry实例化时的操作：
1.调用父类的构造方法，确保父类先完成实例化；
2.连接zookeeper，并添加状态监听器，当状态事件为 重新连接 时，会把之前注册的服务和订阅的服务放在失败重试的集合中，等待父类FailbackRegistry定时任务重试；(因为是临时节点，断开后，zk会把节点删除，因此需要重新注册和订阅服务)
`
`FailbackRegistry实例化时的操作：
1.调用父类的构造方法；
2.ScheduledExecutorService实现定时任务，默认5s执行一次，进行失败重试，扫描失败重试集合，取出对应的任务，执行重试操作，包括 注册服务、取消注册服务、订阅服务、取消订阅服务、推送通知失败notify服务，这些都会进行重试。`
`AbstractRegistry实例化时的操作：
1.加载本地的服务缓存文件，文件中保存了provider相关的数据URL；
2.如果本地缓存文件不为空，则将文件的内容加载到内存中，后续可以使用；
3.通知订阅notify该url的订阅者`

判断引用的服务是否有分组和是否需要合并不同的返回结果，使用默认的分组聚合集群策略。

调用doRefer方法，选择配置或者默认的集群策略。

消费者向注册中心订阅服务(在consumers目录下创建临时节点)，zk收到订阅请求后，会将提供者列表信息全量推送给消费者，消费者收到zk推送过来的消息后，进行服务引用。触发Directory监听器的可以是订阅请求，覆盖策略消息，路由策略消息。
消费者收到消息后，会刷新本地缓存，更新缓存中的服务提供者配置，更新缓存的路由规则配置，然后重建invoker实例。
重建invoker实例的操作：
`1.如果url已经被转换为invoker，则不在重新引用，直接从缓存中获取，注意如果url中任何一个参数变更也会重新引用
2.如果传入的invoker列表不为空，则表示最新的invoker列表
3.如果传入的invokerUrl列表是空，则表示只是下发的override规则或route规则，需要重新交叉对比，决定是否需要重新引用。`

DubboProtocol负责创建DubboInvoker实例，并开启netty服务。

上面介绍了消费者注册到注册中心和订阅注册中心服务。下面是创建服务代理的过程。
## 创建服务代理
AbstractProxyFactory.getProxy(invoker)，根据invoker创建服务代理proxy。
消费者最终使用的也是动态代理创建的代理对象，代理会注册到spring容器中，就可以调用服务方法了。

# debug出现的问题
[Dubbo 服务二次引用问题报告](https://github.com/code4wt/incubator-dubbo-website/blob/bug_report/audit/dubbo_bug_report.md)


