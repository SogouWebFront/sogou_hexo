title: java监控工具(jps,jstat,jstack,jmap,jvisualvm等)
date: 2015-09-15 11:25:40
tags: [java,jvm,shell,linux]
---
对于线上线下服务，针对服务状态，qps，cost等一般都会有一定的监控措施。如果遇到问题，比如cpu占用率高或者程序特别吃内存，对于java程序来说，会用到一些java监控命令和错误定位命令，能够更好的监控服务运行状态，也能够快速定位问题。整理一下我一般使用的命令，下面的命令都是基于oracle hotspot jvm。
<!--more-->

**Monitoring Tools**
* jps: JVM Process Status Tool, 列出指定机器的jvm pid。
* jstat: JVM Statistics Monitoring Tool, jvm的统计监控工具。
* jstatd: JVM jstat Daemon, 没用过。

**Troubleshooting Tools**
* jinfo: Configuration Info for Java, 打印java配置信息。
* jhat: Heap Dump Browser, 根据dump文件进行分析，可以在浏览器中查看。
* jmap: Memory Map for Java, 内存使用情况。
* jsadebugd: Serviceability Agent Debug Daemon for Java, 没用过。
* jstack: Stack Trace for Java, 打印指定线程的栈信息。

**Java Troubleshooting, Profiling, Monitoring and Management Tools**
* jcmd: JVM Diagnostic Commands Tool, 给jvm发送诊断请求。
* jconsole: A JMX-compliant graphical tool for monitoring a Java virtual machine, jvm性能分析，图形界面。
* jmc: Java Mission Control, 没用过。
* jvisualvm: Java VisualVM, 查看程序的详细信息，图形界面。

还有更多的一些工具，具体可以看http://docs.oracle.com/javase/7/docs/technotes/tools/index.html
下面只整理了自己平时用到的，没有用过的暂时不提，等用过之后再补充。

## Monitoring Tools
### 1. jps: `jps [ options ] [ hostid ]`
一般只是为了打印出程序的pid, 可能会用到 -m, -l, -v选项, -m列出main方法的参数, -l列出完整jar包, -v列出jvm的参数。

    ~ jps -m
    27364 Resin --root-directory /usr/local/resin -conf /usr/local/resin/conf/resin.xml -server web -socketwait 33740 -server web -root-directory /usr/local/resin -log-directory /usr/local/resin/log restart
    21126 Jps -m
    22478 WatchdogManager -Xloggc:log/gc.log.2015081910 -server web -root-directory /usr/local/resin -conf /usr/local/resin/conf/resin.xml -log-directory /usr/local/resin/log start --log-directory /usr/local/resin/log
    
    ~ jps -l
    27364 com.caucho.server.resin.Resin
    21147 sun.tools.jps.Jps
    22478 com.caucho.boot.WatchdogManager
    
    ~ jps -v 
    涉及到resin具体配置，不列出具体内容了。

### 2. jstat: `jstat [ generalOption | outputOptions vmid [interval[s|ms] [count]] ]`
先看一下有哪些统计项可选
    
Option|Displays
------------- |:-------------
class | Statistics on the behavior of the class loader.
compiler | Statistics of the behavior of the HotSpot Just-in-Time compiler.
gc | Statistics of the behavior of the garbage collected heap.
gccapacity | Statistics of the capacities of the generations and their corresponding spaces.
gccause | Summary of garbage collection statistics (same as -gcutil), with the cause of the last and current (if applicable) garbage collection events.
gcnew | Statistics of the behavior of the new generation.
gcnewcapacity | Statistics of the sizes of the new generations and its corresponding spaces.
gcold | Statistics of the behavior of the old and permanent generations.
gcoldcapacity | Statistics of the sizes of the old generation.
gcpermcapacity | Statistics of the sizes of the permanent generation.
**gcutil** | **Summary of garbage collection statistics.**
printcompilation | HotSpot compilation method statistics.

最常用到是gcutil, `jstat -gcutil ${pid} 1000 100`, 每1000ms打印一次gc统计情况，一共打印100次。 
    
    ~ jstat -gcutil 29209 1000 100
    S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
    17.56   0.00  88.08  36.98  97.54  96.04  12166  309.541    12    4.174  313.715
    17.56   0.00  88.08  36.98  97.54  96.04  12166  309.541    12    4.174  313.715
    17.56   0.00  88.53  36.98  97.54  96.04  12166  309.541    12    4.174  313.715
    17.56   0.00  88.53  36.98  97.54  96.04  12166  309.541    12    4.174  313.715
    17.56   0.00  88.54  36.98  97.54  96.04  12166  309.541    12    4.174  313.715
    17.56   0.00  88.55  36.98  97.54  96.04  12166  309.541    12    4.174  313.715
    17.56   0.00  89.35  36.98  97.54  96.04  12166  309.541    12    4.174  313.715
    ...

先看一下打印出来的都是什么： 

Column | Description
------------- |:-------------
S0 | Survivor space 0 的利用率
S1 | Survivor space 1 的利用率
E | Eden space 的利用率
O | Old space 的利用率
P | Permanent space 的利用率
YGC | young gc数量
YGCT | young gc时间
FGC | Full gc数量
FGCT | Full gc时间
GCT | 所有的gc时间, (YGCT + FGCT)


对java开发工程师来说，这些应该都不陌生。如果在上线过程中，我一般会比较关注YGC和FGC, 如果发现gc过于频繁，会去查看内存的使用情况。
jvm内存管理可以看一下[探秘java虚拟机--内存管理与垃圾回收](http://www.blogjava.net/chhbjh/archive/2012/01/28/368936.html)，不过是基于jdk1.6的，对于java8，可以看一下[java8的元空间](http://itindex.net/detail/49579-java-%E7%A9%BA%E9%97%B4)。

## Troubleshooting Tools
### 1. jinfo: `jinfo [ option ] pid` 
除了可以根据pid之外，还可以根据core-file或远程地址。
可以查看jvm的配置信息，类似于java程序中`System.getProperties()`，一般会在查看encoding的时候使用。

    ~ jinfo 29209
    Attaching to process ID 29209, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 25.45-b02
    Java System Properties:
    
    java.runtime.name = Java(TM) SE Runtime Environment
    java.vm.version = 25.45-b02
    sun.boot.library.path = /usr/lib/jvm/java-1.8.0-oracle-1.8.0.45.x86_64/jre/lib/amd64
    javax.management.builder.initial = com.caucho.jmx.MBeanServerBuilderImpl
    java.vendor.url = http://java.oracle.com/
    java.vm.vendor = Oracle Corporation
    ...
    
### 2. jmap: `jmap [ option ] pid`
除了可以根据pid之外，还可以根据core-file或远程地址。
* `jmap -heap ${pid}`，可以列出堆的配置和使用情况，如果gc频率过高或内存占用太高，能够从这些信息中找到是哪一块出了问题。
* `jmap -histo[:live] ${pid}`，可以查看堆内存中的对象数目、大小统计直方图，如果带上live则只统计活对象。


    ~ jmap -histo:live 29209 | more 
    num     #instances         #bytes  class name
    ----------------------------------------------
    1:       3040709      585742624  [C
    2:        110636      379989312  [B
    3:       4797182      115132368  java.util.LinkedList$Node
    4:        121324       81142520  [Ljava.lang.Object;
    5:         19398       78996664  [I
    6:       2689135       64539240  java.lang.String
    7:          1443       47278640  [Lorg.apache.xpath.objects.XObject;
    8:        397818       38190528  org.apache.xpath.axes.AxesWalker
    9:        917760       29368320  com.caucho.env.actor.ValueActorQueue$ValueItem
    10:       708201       22662432  java.util.HashMap$Node
    11:          445       19404664  [J
    12:       363053       11617696  java.util.LinkedList
    15:        82855        9279760  org.apache.xpath.axes.WalkingIterator
    16:        60086        7210320  org.apache.xalan.templates.ElemLiteralResult
    17:        71974        6909504  org.apache.xalan.templates.ElemTextLiteral
    18:        65536        6291456  com.caucho.server.dispatch.Invocation
    19:        12072        5723592  [Ljava.util.HashMap$Node;
    20:        60593        5332184  org.apache.xalan.templates.ElemValueOf
    21:       190114        4562736  java.util.ArrayList
    ...

* `jmap -dump:format=b,file=dumpFileName ${pid}`，jmap还可以把进程的内存使用情况dump到文件中，然后用jhat或jvisualvm分析。**对于线上环境不要轻易使用！jmap -dump:live每次都会触发一次Full GC**
dump出来的文件一般都会很大，jhat分析时可能需要加上-J-Xmx512m这种参数来指定最大堆内存。


    ~ jmap -dump:live,format=b,file=jmapdump 2106
    Dumping heap to /tmp/jmapdump ...
    Heap dump file created
    ~ jhat -port 9998 jmapdump
    
jhat进行分析，并启动一个server，可以在浏览器中查看。图片中只截取了很小一部分。
![jmap dump](http://7xlgd1.com1.z0.glb.clouddn.com/images/io/jmap-dump.png)

### 3. jstack: `jstack [ option ] pid`
除了可以根据pid之外，还可以根据core-file或远程地址。
jstack可以定位到线程堆栈，根据堆栈信息我们可以定位到具体代码。如果java程序崩溃生成core文件，jstack工具可以用来获得core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。另外，jstack工具还可以附属到正在运行的java程序中，看到当时运行的java程序的java stack和native stack的信息, 如果现在运行的java程序呈现hung的状态，jstack是非常有用的。

在实际应用中，一般会通过ps, printf, top, jstack来查找出最耗cpu的java线程并定位到某一行代码。
下面举个实例：
1. 找到对应进程id, 使用`ps -ef | fgrep resin | fgrep -v fgrep`，找到resin的pid，或者直接使用`jps`。
1. 找出resin进程中最耗cpu的线程，可以使用`ps -Lfp ${pid}`或`top -Hp ${pid}`或`ps -mp ${pid} -o THREAD,tid,time`，我一般使用`top -Hp ${pid}`, `top`命令中根据某一行排序使用"F"，然后选择对应项前的a-z即可。
![top-Hp](http://7xlgd1.com1.z0.glb.clouddn.com/images/io/top-hp.png)
1. 可以看到cpu使用时间最高的是31255, 我们要根据这个pid得到其16进制数，也就是nid


    ~ printf "%x\n" 31255
    7a17
    
1. 然后使用jstack打印进程的堆栈信息，再根据nid进行grep。


    ~ jstack 29209 | fgrep -A10 7a17
    "vr_cache(Recver3)" #119 prio=5 os_prio=0 tid=0x00007f6a3007b000 nid=0x7a17 runnable [0x00007f6abc5f6000]
    java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
        at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
        at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:79)
        at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
        - locked <0x00000006d70100c8> (a sun.nio.ch.Util$2)
        - locked <0x00000006d70100b0> (a java.util.Collections$UnmodifiableSet)
        - locked <0x00000006d7009958> (a sun.nio.ch.EPollSelectorImpl)
        at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
        at common.connectionpool.async.Receiver.run(Receiver.java:379)
可以看到CPU消耗都花在了epollWait上，再深究下去，是自己实现的代码中Receiver类中第379行引起的，我们看一下代码：
    
    int num = this.selector.select(pool.robinTime);

这一行正是selector进行等待的时候，对应于epoll来说就是epoll_wait函数。定位到问题，可以去找对应方案了。
如果对IO多路复用机制的可以看一下相关文章，主要有select, poll, epoll。

## java图形化界面监控工具
jconsole和jvisualvm都需要图形化界面，都可以连接远程server查看。
图形化界面看起来都比较容易，在这不做赘述了。
### 1. jconsole: 
![jconsole overview](http://7xlgd1.com1.z0.glb.clouddn.com/images/io/jconsole-overview.png)

### 2. jvisualvm
![jvisualvm monitor](http://7xlgd1.com1.z0.glb.clouddn.com/images/io/jvisualvm-monitor.png)

## 参考
* [oracle官方文档](http://docs.oracle.com/javase/8/)
* [jdk tools and utilities](http://docs.oracle.com/javase/7/docs/technotes/tools/index.html)
* [探秘java虚拟机--内存管理与垃圾回收](http://www.blogjava.net/chhbjh/archive/2012/01/28/368936.html)
* [java8的元空间](http://itindex.net/detail/49579-java-%E7%A9%BA%E9%97%B4)
* [select、poll、epoll之间的区别总结(整理)](http://www.cnblogs.com/Anker/p/3265058.html)
* [jmap -dump:live为啥会触发Full GC](http://langzi-xl.iteye.com/blog/798905)