---
layout: post
title: java out-of-memory 调优 学习笔记一
category: Recommendation-algorithm
tags: [Recommendation-algorithm]
---
今天一个工厂的一个服务突然报OOM，是一个windows的server，部署了很多服务。找了一下原因，记录一下。 

增加配置加大程序最大内存。用-XX:-UseGCOverheadLimit选项来关闭GC Overhead
limit exceeded

## 默认堆大小
你也可以在程序里试试打印 Runtime.getRuntime().maxMemory() 的值 看看是多少
官网说明： [https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size]()

除非在命令行中指定了初始和最大堆大小，否则将根据计算机上的内存量计算它们。

### 客户端JVM默认初始和最大堆大小
默认的最大堆大小是物理内存的一半，直到物理内存大小为192兆字节（MB），否则为物理内存的四分之一，物理内存大小为1千兆字节（GB）。

例如，如果您的计算机具有128 MB的物理内存，则最大堆大小为64 MB，并且大于或等于1 GB的物理内存会导致最大堆大小为256 MB。

除非程序创建足够的对象来要求它，否则JVM实际上不会使用最大堆大小。在JVM初始化期间分配一个小得多的数量，称为初始堆大小。此数量至少为8 MB，否则为物理内存的1/64，最大物理内存大小为1 GB。

分配给年轻代的最大空间量是总堆大小的三分之一。

### 服务器JVM默认初始和最大堆大小
要用 java -server <类名> 来启动你的程序 ..

默认的初始和最大堆大小在服务器JVM上的工作方式与在客户端JVM上的大小相同，只是默认值可以更高。在32位JVM上，如果有4
GB或更多的物理内存，则默认的最大堆大小可以高达1 GB。

在64位JVM上，如果存在128GB或更多物理内存，则默认最大堆大小最多可为32GB。

可以通过直接指定这些值来设置更高或更低的初始和最大堆;见下一节。

### 指定初始和最大堆大小
可以使用标志 -Xms（初始堆大小）和 -Xmx（最大堆大小）指定初始和最大堆大小。

如果知道应用程序需要多少堆才能正常工作，则可以设置-Xms并-Xmx使用相同的值。

如果没有，JVM将首先使用初始堆大小，然后增加Java堆，直到它在堆使用和性能之间找到平衡。

其他参数和选项可能会影响这些默认值。要验证默认值，请使用该-XX:+PrintFlagsFinal选项并MaxHeapSize在输出中查找。

例如，在Linux或Solaris上，您可以运行以下命令：

            java -XX：+ PrintFlagsFinal <GC options> -version | grep MaxHeapSize

还有一种OOM的可能Linux下发生OOM，不一定是因为Java服务耗内存，也可能是因为其他程序申请了很多内存，此时所有应用所需要的内存超过物理内存，然后Java服务很耗内存且被Linux操作系统找到，就会被
kill，这是Linux为避免物理内存过载导致系统崩溃而采取的内存保护机制，这种机制称为OOM
Killer，具体原因参考文末参考资料中的Linux 下的 OOM
Killer部分。之前工作中遇到过ElasticSearch数据存储服务和Fluentd日志采集服务部署在同一台服务器上，Fluentd内存泄漏导致的ElasticSearch服务被kill的情况。后来是review了Fluentd的代码，解决了内存泄露问题，并将其与ElasticSearch服务分开部署解决。

## GC Overhead Limit Exceeded Error简介
简单地说，Garbage Collection (GC)就是JVM回收不再使用的对象，释放内存的过程。GC Overhead Limit Exceeded error是java.lang.OutOfMemoryError家族的一员，表示JVM内存被耗尽。接下来看看引起java.lang.OutOfMemoryError: GC Overhead Limit Exceeded错误的原因是什么，以及如何解决这个错误。

OutOfMemoryError是java.lang.VirtualMachineError的子类，当JVM资源利用出现问题时抛出，更具体地说，这个错误是由于JVM花费太长时间执行GC且只能回收很少的堆内存时抛出的。根据Oracle官方文档，默认情况下，如果Java进程花费98%以上的时间执行GC，并且每次只有不到2%的堆被恢复，则JVM抛出此错误。换句话说，这意味着我们的应用程序几乎耗尽了所有可用内存，垃圾收集器花了太长时间试图清理它，并多次失败。

在这种情况下，用户会体验到应用程序响应非常缓慢，通常只需要几毫秒就能完成的某些操作，此时则需要更长的时间来完成，这是因为所有的CPU正在进行垃圾收集，因此无法执行其他任务。

### 解决方案
理想的解决方案是通过检查可能存在内存泄漏的代码来发现应用程序所存在的问题，这时需要考虑：

    应用程序中哪些对象占据了堆的大部分空间？
    （What are the objects in the application that occupy large portions of the heap?）
    
    这些对象在源码中的哪些部分被使用？
    （In which parts of the source code are these objects being allocated?）

我们还可以使用自动化图形工具，比如 [JVisualVM](https://visualvm.github.io/) 、 [JConsole](https://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html) ，它可以帮助检测代码中的性能问题，包括java.lang.OutOfMemoryError。

最后一种方法是通过更改JVM启动配置来增加堆大小，或者在JVM启动配置里增加-XX:-UseGCOverheadLimit选项来关闭GC Overhead limit exceeded。例如，以下JVM参数为Java应用程序提供了1GB堆空间：

        java -Xmx1024m com.xyz.TheClassName

以下JVM参数不仅为Java应用程序提供了1GB堆空间，也增加-XX:-UseGCOverheadLimit选项来关闭GC Overhead limit exceeded：

        java -Xmx1024m -XX:-UseGCOverheadLimit com.xyz.TheClassName

但增加-XX:-UseGCOverheadLimit选项的方式治标不治本，JVM最终会抛出java.lang.OutOfMemoryError: Java heap space错误。
总之，如果实际的应用程序代码中存在内存泄漏，那么以上列举的方法并不能解决问题，相反，我们将推迟这个错误。因此，更明智的做法是彻底重新评估应用程序的内存使用情况。

## JAVA -XX 参数介绍及调优

[link](https://blog.csdn.net/earbao/article/details/103495508)



## 参考

* [OutOfMemoryError: GC Overhead Limit Exceeded错误解析](https://blog.csdn.net/github_32521685/article/details/89953796)
* [java-xx参数介绍及调优总结](https://blog.csdn.net/earbao/article/details/103495508)
* [基于内容推荐算法详解](https://blog.csdn.net/nicajonh/article/details/79657317)
* [Oracle官方总结的OOM异常及处理方法](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html)
* [Linux下的OOM Killer](https://www.vpsee.com/2013/10/how-to-configure-the-linux-oom-killer/)
* [什么情况下发生OOME](https://blog.csdn.net/jiangxiulilinux/article/details/105345410)