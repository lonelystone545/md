[TOC]
# logback加载
当我们使用logback-classic.jar时，应用启动，那么logback会按照如下顺序进行扫描：

在系统配置文件System Properties中寻找是否有logback.configurationFile对应的value
在classpath下寻找是否有logback.groovy（即logback支持groovy与xml两种配置方式）
在classpath下寻找是否有logback-test.xml
在classpath下寻找是否有logback.xml
以上任何一项找到了，就不进行后续扫描，按照对应的配置进行logback的初始化，具体代码实现可见ch.qos.logback.classic.util.ContextInitializer类的findURLOfDefaultConfigurationFile方法。

当所有以上四项都找不到的情况下，logback会调用ch.qos.logback.classic.BasicConfigurator的configure方法，构造一个ConsoleAppender用于向控制台输出日志，默认日志输出格式为"%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"。

# logback的configuration

logback的重点应当是Appender、Logger、Pattern，在这之前先简单了解一下logback的<configuration>，<configuration>只有三个属性：

scan：当scan被设置为true时，当配置文件发生改变，将会被重新加载，默认为true
scanPeriod：检测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认为毫秒，当scan=true时这个值生效，默认时间间隔为1分钟
debug：当被设置为true时，将打印出logback内部日志信息，实时查看logback运行信息，默认为false

参考[logback详解](https://zhuanlan.zhihu.com/p/36285566)
