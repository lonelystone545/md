# 参考
[Dubbo可扩展机制实战](http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi.html)：LoadBalance就是一个SPI，我们可以实现LoadBalance接口自定义负载均衡的策略。
[Dubbo中SPI扩展机制详解](https://cxis.me/2017/02/18/Dubbo%E4%B8%ADSPI%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/)
[Dubbo中SPI源码解析](https://cxis.me/2019/02/20/Dubbo%E4%B8%ADSPI%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)

# 和JAVA SPI比较
1. JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。Dubbo则根据name来得到对应的扩展类。从ClassPath下META-INF文件夹下读取扩展点配置文件。
2. 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。
3. 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 getName() 获取脚本类型的名称，但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和 ruby 对应不起来，当用户执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。
4. Dubbo的扩展点支持默认实现，比如，Protocol中的@SPI("dubbo")默认为DubboProtocol，默认扩展点可以通过ExtensionLoader.getExtensionLoader(Protocol.class).getDefaultExtension()获取
5. Dubbo的扩展点可以动态获取扩展对象，比如：ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName)来获取扩展对象，我觉得这是Dubbo扩展点设计的很有意思的地方，非常的灵活方便，代码中大量的用到了这个方法。原理：在加载扩展点的实现的时候，并不是调用具体的实现逻辑找到实现类，而是在注入扩展点时，不需要知道这个扩展点的具体实现是什么，因此会注入一个自适应的实现，那么等到运行的时候能够根据参数进行动态的选择，从而调用真正的实现。（这样在配置中心做修改的时候，最终修改的是url参数，那么就可以获得不同的实现）
