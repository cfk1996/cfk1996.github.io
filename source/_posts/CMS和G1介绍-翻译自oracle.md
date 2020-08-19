---
title: CMS和G1介绍(翻译自oracle)
date: 2020-08-15 11:14:40
tags:
    - java
---

我们的游戏服务器用的是G1垃圾回收器，在学习的过程中找到一篇oracle的官方教程，觉得还不错，所以一边学习一边翻译一下，官方教程链接在下面。

[Getting Started with the G1 Garbage Collector](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)


# 概览

## 目标

这篇教程包括了如何在jvm中使用G1回收器的基础知识，并且可以学到G1的内部运行机制，关键的控制参数和一些gc日志基础。

## 耗时

1小时

## 介绍

文章介绍了g1的基础。在第一部分，介绍了JVM的整体概览结构，其中就包括垃圾回收器和一些性能相关的机制。接下来的一部分内容介绍了CMS垃圾回收器是如何在`Hotspot jvm`中工作的。再接下来，一步步的介绍g1是如何在hotspot jvm中运行的;紧接着是一些使用g1回收器时的关键jvm参数介绍，最后你会学到一些与g1相关的日志内容。

跳过`硬件与软件要求`和`java language，jre，jdk,jvm`等基础内容介绍

<!--more-->

# 探索jvm架构

## Hotspot架构

**(Hotspot jvm 全部用jvm代替)**

JVM的架构支持强大的基础特性和能力支持，提供了实现高性能和大规模拓展性的能力。比如说JIT编译器在运行时生成高性能的本地机器指令来提高性能。除此之外，通过不断的优化运行时环境和多线程的垃圾回收功能，即使在最大的可用计算机系统里jvm也提供了高的伸缩性。

![jvm架构](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/jvm-Architecture.png)

jvm的主要构成组件包括类加载器，运行时数据存储区和执行引擎。

## 关键组件

jvm的组件中有三个是关注于性能调优的(heap, garbage collector, jit)。heap是对象存储的地方，这块区域会被垃圾回收器管理，大多数调优的方式都与调整堆大小和选择最适合程序状况的垃圾收集器有关。JIT编译器对性能也有很大的影响，但是这是随着jvm的版本更新的。

# 性能基础

通常对java应用调优时关注两个主要的指标：响应速度或者吞吐量。

- 响应速度：应用能以多快的速度响应一个请求
    + 网点返回一个网页的时间
    + 一次数据库查询的时间
    + ...

对注重响应时间的应用来说，长时间的停顿时间是不可接受的，必须在很短的时间内响应请求

- 吞吐量：在一段时间内，处理的请求数量
    + 一段时间内完成的事务数量
    + 一小时内数据库可以完成的查询次数
    + ...

对注重吞吐量的应用来说，更关注一段时间内的工作总量，偶尔的单个请求耗时长点是可以接受的

# g1垃圾回收器

g1垃圾回收器是用于服务端的，是针对大内存的多处理器的应用。它可以设立一个最大停顿时间，并且有很高的可能性达到这个目标同时也可以达到很大的吞吐量。g1垃圾收集器在JDK7 update4及之后的版本都已经支持了。g1收集器是为了这样的应用设计的：

1. 可以像CMS一样与应用线程并行
1. 压缩释放的内存并且不增加gc引起的暂停时间
1. 需要可预测的gc停顿持续时间
1. 不想丢失大量的吞吐量性能
1. 不想增加堆的大小

g1计划用于长期替换cms。对比g1和cms，有很多的不同让g1成为一个更好的选择。其中一个是g1是一个可以压缩内存的收集器，g1堆空间的压缩紧密并且划分为更多的region，可以避免细粒度的空闲队列来分配内存。这大大的简化了垃圾收集器，并且解决了内存碎片的问题。同时g1提供了更加可控的gc停顿时间，允许用户指定它们期望的停顿目标。

## g1运行概述

旧的垃圾收集器(serial, parallel, CMS)都将堆划分为大小固定的三块区域：年轻代，老年代和永久代。

![HeapStructure](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/HeapStructure.png)

所有对象最终都属于这三个部分之一。

但是g1使用了一种不同的方式

![g1heap](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/g1heap.png)

堆被划分为一堆大小相等的区域，每一个都是一片连续的虚拟内存。尽管这些region还是像旧收集器一样分成三种状态(eden,survivor,old)，但是它们的大小不是固定的，这提供了更加灵活的内存使用方式。

g1执行垃圾回收的过程与cms很相似。G1首先进行一次并发的全局标记过程来决定堆中活的对象。标记过程结束后，G1知道哪些region大部分是空的，然后它首先收集这些区域并通常可以释放很多的空闲空间；这也是为什么G1被称之为垃圾优先收集器。顾名思义，G1专注收集和压缩那些充满可回收对象(即垃圾对象)的区域。G1使用一种叫做停顿预测模型来达到用户设定的停顿时间目标，并基于这个时间只选取一部分区域进行回收。

被G1认定为可以回收的区域通过`evacuation`(不会翻译)的方式来收集，G1从一个或多个这样的区域中把对象复制到堆中其他的一个region中，并且在这个过程中会压缩和释放内存。在多处理器的系统中，这步操作是并行的，可以减少停顿时间和增加吞吐量。因此随着每一次的垃圾回收，G1在用户设置的时间内不断的减少碎片。相比于cms和paralleold，G1表现出的能力更加优秀。CMS不能够压缩内存，ParalleOld只能够压缩整个heap，导致来不可控的停顿时间。

必须注意到G1并不是一个实时的收集器。它以很高的可能性去满足用户设定的停顿时间要求，但是不是一定可以达到这个目标。基于上一次垃圾回收的数据，G1会估计在用户指定时间内可以回收多少个region。因此，这个收集器拥有收集一个区域耗时的合理的统计模型，并决定选择哪一个和选择多少个region来执行回收。

> G1有与应用线程并发的阶段(refinement,marking,cleanup)，也有并行的(多处理器，stop the world)阶段。full gc依然是单线程的，但是应该对应用调优来避免full gc。

## G1 footprint(脚印？)

如果你的应用从ParalleOld或者cms转换到G1，你很有可能发现内存变大了，这很大程度上与G1用到的“记数”数据结构有关，比如`Remembered set`和`Collection set`。

`Remembered Set`记录自己的区域内的对象被其他区域的对象引用，每个region都有这样的set。Rset可以提供并行和独立收集一个区域的能力，Rset导致的内存变化的影响在5%内。

`Collection Set`记录在一次gc中将要被回收的区域，所有在CSet中的活对象都会在gc中被擦除或者复制移动。CSet对内存的影响小于1%。

## 推荐使用G1的情景

G1最关注的是提供一种大内存且低GC延迟的方法。这意味着大于等于6GB的堆内存，可以稳定的可预测的将停顿时间控制在小于0.5秒的时间内。

如果现在有应用使用ParalleOld或者CMS收集器，并且它们具有如下这些特性，将收集器转换为G1是有帮助的：

1. full gc太频繁或者持续时间久
1. 对象分配律或者提升率差异大
1. 不希望长时间的gc收集和压缩耗时(大于0.5秒)

如果跑的好好的，也可以不更换为G1.

# 回顾CMS的gc过程

CMS回收old区域时通过将大多数回收工作变成与应用线程并发的方式来减小垃圾回收的停顿耗时。一般来说，CMS不会复制和压缩活的对象，如果出现碎片化的问题，那么就需要分配更大的内存。

NOTE: CMS在年轻代使用与parallel相同的算法

## CMS收集阶段

CMS收集器回收老年代时有如下这些过程:

1. 初始标记(stop the world)
    标记老年代的活对象，包括那些被年轻代对象访问的对象，这个时间相对来说比较短。
1. 并发标记阶段
    在应用线程执行时，并发的遍历寻找老年代的可达对象。然后从已标记的对象开始扫描，并从根开始标记所有可访问对象。CMS过程中分配的对象会被立马标记为活的
1. 再标记阶段(stw)
    寻找那些在并发标记中漏掉的对象
1. 并发清除
    回收标记阶段找到的不可达对象，回收死亡对象时把这些对象的内存空间加到空闲链表(用于后续的内存分配)，这个过程可以合并空间空间。需要注意的是：这个过程活的对象并没有被移动
1. 重置阶段
    清除gc过程产生的数据结构，准备下一次gc

## 回顾CMS垃圾收集的过程

接下来，一步步的回顾垃圾收集的过程。

1. CMS收集器的堆结构

堆被划分为三块区域。

![cms堆结构](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/cmsHeap.png)
    
年轻代被分为一个eden区和两个survivor区。老年代是一片连续的空间，垃圾收集是在原地发生的。除非发生full gc，不然不会发生压缩。

2. CMS的年轻代回收机制

年轻代用绿色标记，老年代用蓝色。下图是程序运行一段时间后，内存可能的状态。对象零散的分散在老年代。

![youngGC](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/youngGC.png)

使用CMS时，老年代在原地被回收，它们不会被移动，这些堆空间不会发生压缩，除非发生一次full gc。

3. 年轻代回收

eden区和survivor区的活对象会被复制到另外一个survivor区。那些老对象达到年龄阈值时会晋升到老年代。

![年轻代回收](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide3.png)

4. 年轻代回收后

年轻代回收后，eden区域和其中一个survivor会被清空。

![after young gc](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide4.png)

新晋升的对象用深蓝色标记。绿色的对象是在年轻代中存活下的但是没有晋升的对象。

5. CMS老年代回收

CMS会有两个stop the world阶段：初始标记和再标记阶段。当老年代的占用内存达到一个比例时，CMS就开始了。

![老年代回收](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide5.png)

(1)初始化标记是一个很短的停顿，活的对象会被标记(2)并发标记阶段，在应用线程运行时寻找活的对象(3)在重标记阶段，寻找在第二个阶段被漏掉的活对象

6. 并发清除阶段

那些在前几个阶段没有被标记的对象会被回收，但没有被压缩。
(图片与上一个一样)

7. 擦除后

在擦除阶段后，可以看到有很大的一部分内存被释放了，也可以看到内存没有被压缩。

![擦除后](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide7.png)

最后CMS会进入第五阶段，清除一些数据结构然后准备下一次老年代内存占用达到阈值。

# 一步步介绍G1的过程

G1使用了一种不同的方式来分配堆

1. G1堆结构

堆被划分为很多固定大小的区域，区域的大小是在jvm启动时设定的，通常来说，jvm会划分大概2000个区域，区域的大小1到32MB.

2. G1堆分配

事实上，这些区域在逻辑上被映射为eden, survivor和old三种角色。

![g1heap](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/g1heap.png)

区域的颜色代表来它所属的角色。活的对象会从一个区域复制和移动到另一个区域。区域的回收有stw和非stw阶段。

除了上面展示的三种角色，还有第四种被称作大对象的区域。这些区域是用来存放那些比区域大小的一半还大的对象，这些区域里剩下的空间将不会被使用到。

NOTE:在这篇文章发表时，收集大对象还有被优化，因此需要避免创建大对象。

3. G1年轻代

堆被划分为大约2000个区域，大小可以是1MB到32MB这个范围。蓝色区域存储老年代对象，绿色区域存储年轻代对象，老年代并不需要像CMS那样是连续的。

![g1年轻代](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide10.png)

4. 一次年轻代gc

活对象会被复制和移动到一个或多个survivor区域。如果达到年龄阈值，一些对象会晋升到老年代区域。

![一次young gc](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide11.png)

这是一个stw的停顿。为下一次gc计算eden的大小和survivor的大小。计数信息会被用来计算来满足用户设定的停顿时间

5. 年轻代gc结束后

活对象已经被复制到幸存区或者老年区。

![年轻代gc](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide12.png)

可以用下面这些来总结下G1的年轻代。

1. 堆被划分为很多区域
1. 年轻代是一部分不连续的区域的组合。这样可以很容易将年轻代的大小改变
1. 年轻代垃圾回收会stw，所有应用线程都会停止
1. 垃圾回收是多线程并发的
1. 活对象被复制到新的survivor区域或者old区。

# G1的老年代垃圾回收

像CMS收集器一样，G1被设计为一个老年代低停顿时间的收集器。下面的表格介绍了G1回收老年代的几个阶段。

1. 初始化标记(stw)
    这是一次stw活动，会伴随着一次young gc。标记幸存者区域，这个区域可能有对老年代的引用
1. 根区域扫描
    扫描幸存者区寻找对老年代的引用。这个过程与应用线程一起运行。
1. 并发标记
    在整个堆中寻找活对象。与应用线程一起运行。这个阶段可以被young gc打断。
1. 再标记(stw)
    完成对堆中活对象的标记。使用了一种叫做`snapshot-at-the-beginning (SATB)`的算法，比CMS中的算法要快的多
1. 清除阶段(stw和并发都有)
    + 对活对象和完全的空闲区域计数(stw)
    + 清除Remembered Sets(stw)
    + 重置空的区域并把它们放入空闲链表(并发)
*. 复制阶段(stw)
    这是停顿应用线程来擦除和复制活对象到没有使用过的区域的阶段。如果在年轻代发生，那么gc日志记录为`[GC pause (young)]`，如果在年轻代和老年代都发生了，gc日志记录为`[GC Pause (mixed)]`

## 一步步分析G1老年代垃圾回收

上面定义了G1老年代回收的几个阶段，下面看下它们如何与老年代交互。

1. 初始化标记

对活对象的初始化标记伴随着一次young gc，在日志里被记录为`GC pause (young)(inital-mark)`

![初始化标记](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide13.png)


2. 并发标记阶段

如果空的区域(图中打X的区域)被找到了，它们会在再标记阶段立马被清除。同时也会计算决定活跃度的信息。

![并发标记阶段](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide14.png)

3. 再标记阶段

空区域会立马被回收，现在可以计算所有区域的活跃度信息。

![再标记阶段](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide15.png)

4. 复制/清除阶段

G1选出最低活跃度的一些区域，因为这些区域可以被最快速的回收。然后这些区域会随着一次young gc一起被回收。在gc日志里这被记录为`[GC pause (mixed)]`。所以年轻代和老年代都会一起被回收。

![复制清除](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide16.png)

5. 复制清除后

被选中的低活跃度的区域被回收和压缩成图中深蓝色和深绿色的区域。

![复制清除后](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/Slide17.png)

## 总结g1老年代GC

1. 并发标记阶段
    + 活跃度信息会在应用线程运行时并发的被计算
    + 区域的活跃度信息决定了在复制清除阶段哪些区域是最适合被回收的
    + 没有像CMS那样的清除阶段
2. 再标记阶段
    + 使用一种远远好于CMS的、叫做Snapshot-at-the-Beginning (SATB)的算法
    + 完全空闲的区域会被回收
3. 复制/清除阶段
    + 年轻代和老年代一起被回收
    + 老年代的区域根据活跃度信息来选择


# 命令行选项和最佳实践

这一章节会介绍G1的一些命令行选项。

## 基础命令行参数

开启G1垃圾收集器: `-XX:+UseG1GC`

下面有一段启动demo程序的jvm参数。

`java -Xmx50m -Xms50m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar c:\javademos\demo\jfc\Java2D\Java2demo.jar`

## 关键命令行开关

`-XX:MaxGCPauseMillis=200` 设置最大gc停顿时间。JVM会尽量达到这个条件，所以有时候停顿时间可能会超过设置的目标。默认值是200毫秒。

`-XX:InitiatingHeapOccupancyPercent=45` 当对象占用达到**整个堆**的45%时，启动一次老年代GC。如果这个值为0，则进行常量的gc循环。默认值是45.

## 最佳实践

1. 不要设置年轻代的大小

当明确的通过`-Xmn`设置年轻代的大小时，G1收集器就不会响应我们设置的停顿时间目标；G1也不能够根据需要来扩大或者收缩年轻代的空间，因为年轻代的大小被设定为固定值

2. 响应时间指标

相比于将响应平均时间(ART)设置为`-XX:MaxGCPauseMillis`的参数，可以把这个值设置为目标响应时间的90%或者更多。这意味着90%的用户可以在设定的时间内完成一次请求。记住，JVM不能保证达到这个停顿时间。

3. Evacuation(疏散？)失败

无论GC过程发生在生存者还是晋升对象，当JVM的堆区域用完时，会抛出一次晋升失败。因为区域个数已经最大来，所以堆无法拓展。当使用了`-XX:+PrintGCDetails`参数时，GC日志里会为这种情况标记为`to-space overflow`。

这是一个昂贵的过程：

+ GC还得继续，所以空间必须被释放
+ 复制失败的对象必须保存在原地
+ 任何Cset中保存的区域的RSet的更新都需要重新生成
+ 所有这些操作都很昂贵

4. 如何避免疏散失败

+ 增大堆大小
    - 增大`-XX:G1ReservePercent=n`，默认值是10
    - G1通过保留一部分空间来设定堆的最大值，以防`to space`发生
+ 更早的开始GC周期
+ 通过设置`-XX:ConcGCThreads=n`参数来使用更多的线程进行GC

## 关键jvm参数列表

![](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2020/g1_switches.png)


# GC日志

最后一个话题是通过日志信息来分析GC的性能。这部分简单介绍了下一些可以收集数据的参数和日志里会打印的信息。

## 设置日志详细等级

你可以设置三种不同的日志详细等级。

(1) `-verbosegc` 等价于`-XX:+PrintGc`将日志等级设为**好**，示例输出：

```
[GC pause (G1 Humongous Allocation) (young) (initial-mark) 24M- >21M(64M), 0.2349730 secs]
[GC pause (G1 Evacuation Pause) (mixed) 66M->21M(236M), 0.1625268 secs]    
```

(2) `-XX:+PrintGCDetails`将日志等级设置为**更好**，它输出了下面这些信息：
+ 每个阶段的平均，最小，最大时间
+  Root Scan, RSet Updating (with processed buffers information), RSet Scan, Object Copy, Termination (with number of attempts).(太难翻译了)
+ 还有一些其他耗时：选择CSET的时间，引用处理，引用进队和释放CSet的时间
+ 展示eden, survivors, total heap的占用大小

```
[Ext Root Scanning (ms): Avg: 1.7 Min: 0.0 Max: 3.7 Diff: 3.7]
[Eden: 818M(818M)->0B(714M) Survivors: 0B->104M Heap: 836M(4096M)->409M(4096M)]
```

(3) `-XX:+UnlockExperimentalVMOptions -XX:G1LogLevel=finest`将日志等级设置为**最好**，除了(2)的内容，还包括单个线程的信息。

```
[Ext Root Scanning (ms): 2.1 2.4 2.0 0.0
           Avg: 1.6 Min: 0.0 Max: 2.4 Diff: 2.3]
       [Update RS (ms):  0.4  0.2  0.4  0.0
           Avg: 0.2 Min: 0.0 Max: 0.4 Diff: 0.4]
           [Processed Buffers : 5 1 10 0
           Sum: 16, Avg: 4, Min: 0, Max: 10, Diff: 10]
```

## 确定时间

有一些参数决定了gc日志中的时间如何展示。

(1) `-XX:+PrintGCTimeStamps` jvm启动以来的耗时来展示时间

```
1.729: [GC pause (young) 46M->35M(1332M), 0.0310029 secs]
```

(2) `-XX:+PrintGCDateStamps` 日期形式的时间

```
2012-05-02T11:16:32.057+0200: [GC pause (young) 46M->35M(1332M), 0.0317225 secs]
```

## 理解G1日志

为了让大家理解G1的日志，这部分介绍了一些GC日志的实际输出的术语。下面的例子提供了日志的输出，并对其中的术语和数值进行解释。

NOTE: 更详细的日志介绍可以看[Poonam Bajaj's Blog post on G1 GC logs](https://blogs.oracle.com/poonam/understanding-g1-gc-logs)

这部分比较晦涩，打住了。可以看看上面的链接，详细的介绍了gc日志～