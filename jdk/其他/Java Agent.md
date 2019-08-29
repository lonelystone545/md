[TOC]
# 参考文献
[JVM 源码分析之 javaagent 原理完全解读](https://www.infoq.cn/article/javaagent-illustrated)
[浅谈java agent](http://zhongmingmao.me/2019/01/12/jvm-advanced-agent/)
[谈谈Java Intrumentation和相关应用](http://www.fanyilun.me/2017/07/18/%E8%B0%88%E8%B0%88Java%20Intrumentation%E5%92%8C%E7%9B%B8%E5%85%B3%E5%BA%94%E7%94%A8/)
# java agent
java探针技术，允许开发人员在jvm加载类时或者类运行时，动态的修改字节码文件。
# java agent功能
1. 可以在加载 class 文件之前做拦截，对字节码做修改；(JVM启动后，加载class文件前)
2. 可以在运行期对已加载类的字节码做变更，但是这种情况下会有很多的限制；
3. 小功能：获取所有已经加载过的类/获取所有已经初始化过的类/获取某个对象大小等

# java agent执行流程
JVMTI全称 JVM Tool Interface，是 JVM 暴露出来的一些供用户扩展的接口集合。JVMTI 是基于事件驱动的，JVM 每执行到一定的逻辑就会调用一些事件的回调接口（如果有的话），这些接口可以供开发者扩展自己的逻辑。

比如最常见的，我们想在某个类的字节码文件读取之后、类定义之前修改相关的字节码，从而使创建的 class 对象是我们修改之后的字节码内容，那就可以实现一个回调函数赋给 jvmtiEnv（JVMTI 的运行时，通常一个 JVMTIAgent 对应一个 jvmtiEnv，但是也可以对应多个）的回调方法集合里的 ClassFileLoadHook，这样在接下来的类文件加载过程中都会调用到这个函数中。

JVMTIAgent其实是一个动态库，利用JVMTI暴露出来的接口实现正常情况下无法实现的功能，它一般会实现以下函数：
Agent_OnLoad函数，如果 agent 是在启动时加载的，也就是在 vm 参数里通过 -agentlib 来指定的，那在启动过程中就会去执行这个 agent 里的Agent_OnLoad函数。
Agent_OnAttach函数，如果 agent 不是在启动时加载的，而是我们先 attach 到目标进程上，然后给对应的目标进程发送 load 命令来加载，则在加载过程中会调用Agent_OnAttach函数。
Agent_OnUnload函数，在 agent 卸载时调用，不过貌似基本上很少实现它。

# instrument
[基于Java Instrument的Agent实现](https://www.jianshu.com/p/b72f66da679f)
[☆基于JVMTI的Agent实现](https://www.jianshu.com/p/5cea483f1b36)
java agent实际上就是通过java的instrument(增强)实现的。
使用 Instrumentation，使得开发者可以构建一个独立于应用程序的代理程序（Agent），用来监测和协助运行在 JVM 上的程序，甚至能够替换和修改某些类的定义。有了这样的功能，开发者就可以实现更为灵活的运行时虚拟机监控和 Java 类操作了，这样的特性实际上提供了 一种虚拟机级别支持的 AOP 实现方式，使得开发者无需对 JDK 做任何升级和改动，就可以实现某些 AOP 的功能了。

instrument agent 实现了Agent_OnLoad和Agent_OnAttach两方法，也就是说在使用时，agent既可以在启动时加载，也可以在运行时动态加载。其中启动时加载还可以通过类似-javaagent:myagent.jar的方式来间接加载 instrument agent，运行时动态加载依赖的是 JVM 的 attach 机制（JVM Attach 机制实现），通过发送 load 命令来加载 agent。

java SE 5 中，Instrument 要求在运行前利用命令行参数或者系统参数来设置代理类，在实际的运行之中，虚拟机在初始化之时（在绝大多数的 Java 类库被载入之前），instrumentation 的设置已经启动，并在虚拟机中设置了回调函数，检测特定类的加载情况，并完成实际工作。但是在实际的很多的情况下，我们没有办法在虚拟机启动之时就为其设定代理，这样实际上限制了 instrument 的应用。而 Java SE 6 的新特性改变了这种情况，通过 Java Tool API 中的 attach 方式，我们可以很方便地 在运行过程中动态地设置加载代理类，以达到 instrumentation 的目的。

1. 运行前，类加载的时候修改字节码文件
2. 运行时动态修改
    JDK6提供了Attach API，用来向目标JVM附着（attach）代理工具的程序，开发者能够方便监控一个JVM，运行一个外加的代理程序。
   
    
# JVM Attach
[JVMTI Attach机制与核心源码分析](https://juejin.im/post/5b0d020d518825153f10403f)
Attach机制是JVM提供的一种JVM进程间进行通信的能力，能够让一个进程传命令给另外一个进程，并让他执行内部的一些操作。
`比如，为了让另外一个JVM进程把线程dump出来，那么首先跑了一个jstack的进程，然后传了个pid的参数，告诉它要哪个进程进行线程dump，既然是两个进程，那肯定涉及到进程间通信，以及传输协议的定义，比如：要执行什么操作，传了什么参数等。`
## 实现原理
Attach实现原理是使用了Linux下的文件socket通信。之所以没有采用网络socket，是因为1.是同一个JVM下面的进程进行通信，没有网络开销。2.不需要网络协议的解析、数据包的封装和解封装，提高效率。3.减少对系统资源占用如网络端口。
进程之间通过对文件的读写从而达到信息交互和共享。
## 源码实现
在目标JVM进程启动时，会创建一个signal dispatcher线程，该线程主要用于监听并处理操作系统的信号(linux运行进程间通过信号通信，如kill -9，通知系统强制结束某个进程的生命周期)，该线程会调用os::signal_wait(）方法，从而阻塞当前线程，等待接收系统信号，然后再根据信号做switch的逻辑，对于不同的信号做不同的处理。
但是信号是有限制的，一方面os支持的信号是有限的，二来信号的通信往往是单向的，不方便通信双方进行高效通信。因此在client jvm和target jvm采用socket进行通信。
当我们需要attach到某个目标JVM进程上时，代码如下：

```java
VirtualMachine vm = VirtualMachine.attach(pid);
```
linux通过sem_wait()实现os::signal_wait()，等待信号，当收到信号是sigbreak时，就会触发AttachListener的初始化逻辑，会创建Attach Listener线程，该线程负责接收外部的命令，然后执行该命令并把结果返回给发送者。

## jstack为例进行讲解(jmap)
jstack命令的实现其实是一个叫做JStack.java的类，jstack命令首先会attach到目标JVM进程，产生VirtualMachine类；Linux系统下，其实现类为LinuxVirtualMachine，调用其remoteDataDump方法，打印堆栈信息


# 应用
* 统计方法耗时
    拦截方法，在方法的前后加入计时器功能，用于监控方法耗时，将方法耗时和方法调用情况存入 栈 结构，利用栈 先进后出 的特点对方法调用先后顺序进行处理。
* 日志输出
* debug
    当对java代码进行debug时，其实就是利用 JRE 自带的 jdwp agent 实现的，只是 Eclipse 等工具在没让你察觉的情况下将相关参数 (类似-agentlib:jdwp=transport=dt_socket,suspend=y,address=localhost:61349) 自动加到程序启动参数列表里了，其中 agentlib 参数就用来跟要加载的 agent 的名字，比如这里的 jdwp(不过这不是动态库的名字，JVM 会做一些名称上的扩展，比如在 Linux 下会去找libjdwp.so的动态库进行加载，也就是在名字的基础上加前缀lib，再加后缀.so)，接下来会跟一堆相关的参数，将这些参数传给Agent_OnLoad或者Agent_OnAttach函数里对应的options。
    
* JRebal/Btrace/arthas等工具都是基于Java Agent实现的

# 和其他比较
通过自定义类加载器也能够实现运行时修改代码，但是对代码的侵入性比较高；
通过agent修改字节码，对业务透明，减少入侵。
agent类似于spring aop，不过spring aop需要在源代码基础上做修改，进行增强，而agent可以在不修改源代码上在启动或者运行时对类的字节码进行增强。

`ClassFileTransformer需要实现transform方法，根据自己的功能需要修改class字节码，在修改字节码过程中需要借助javassist进行字节码编辑。
javassist是一个开源的分析、编辑和创建java字节码的类库。通过使用javassist对字节码操作可以实现动态”AOP”框架。
关于java字节码的处理，目前有很多工具，如bcel，asm(cglib只是对asm又封装了一层)。不过这些都需要直接跟虚拟机指令打交道。javassist的主要的优点，在于简单，而且快速，直接使用java编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构，或者动态生成类。`
