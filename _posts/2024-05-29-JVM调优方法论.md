---
layout: post
title:  "JVM调优方法论"
date:   2024-04-01 20:23 +0800
categories: java
typora-root-url: ./image
---

## 一：引言

在软件开发和运维中，JVM作为执行Java程序的核心引擎，扮演着至关重要的角色。

随着应用程序的复杂性和负载不断增加，对JVM的性能和稳定性要求也越来越高。

在此背景下，JVM调优变得至关重要。

JVM调优涉及到一系列的参数设置、垃圾收集器的选择、内存分配策略等方面，对于提高Java应用程序的性能、减少内存泄漏、降低系统崩溃风险都有重要作用。

另外，在大厂面试中，JVM调优的知识也是备受关注的考察点，因为它直接关系到系统的稳定性和性能优化。

候选人对JVM调优的理解和实践能力，可以反映其在Java虚拟机运行机制方面的深度和广度，

需要注意的是，调优并非首选方法，一般而言，解决性能问题的第一步是优化程序本身，只有在必要时才考虑进行JVM调优。

## JVM调优，有什么好处？

JVM调优目的是通过调整Java虚拟机的配置参数、垃圾回收策略和内存分配等手段，提升Java应用程序的性能、稳定性和可靠性。

随着应用规模和用户量的增长，原始的JVM配置可能无法满足业务需求，因此必须进行调优以确保系统的正常运行。

然而，并不是所有异常情况都需要进行JVM调优。

在实际情况中，大多数问题可以通过分析JVM日志文件和业务逻辑来定位，并通过业务层面的优化来解决。

尽管如此，深入了解各项参数和指标仍然至关重要，因为它们有助于更快速地理解和解决问题，调优能带来什么好处？

- **性能层面：**

  通过调整JVM参数和优化垃圾回收机制，能够提高Java应用程序的性能，减少延迟，提升系统响应速度和并发能力、和吞吐量。

- **资源利用：**

  合理配置JVM资源，包括内存、CPU等，能够有效地利用硬件资源，提高系统的资源利用率，降低成本。

- **稳定性：** 通过调优JVM，可减少内存泄漏、OOM（Out of Memory）等问题的发生，提高系统的稳定性和可靠性，降低系统崩溃的风险。

## 二：JVM调优的关注哪些指标？

调优，到底调的是什么？

调优之前，要搞清楚一个问题：怎样才算是“优”。

如何定性？

如何定量？

到底需要其实是需要关注几个关键的指标，以全面评估系统的运行状态和性能表现。

需要有一个具体的指标来衡量性能情况，而在JVM里面衡量性能的两个核心指标分别“吞吐量”和“停顿时间”。

### 核心指标1：吞吐量(throughput)：

程序运行过程中执行两种任务，分别是执行业务代码的 任务 和进行垃圾回收的任务，

吞吐量大，意就是说程序运行业务代码的时间越多， 换句话说，执行业务任务越多， 吞吐量就越高，

吞吐量计算公式 ，

```
吞吐量 = CPU在用户应用程序运行的时间 / （CPU在用户应用程序运行的时间 + CPU垃圾回收的时间），
```

在实践中我们发现对于大多数的应用领域，评估一个垃圾收集(GC)算法如何，有如下一个核心标准：

- 吞吐量越高越好

一般而言GC 的吞吐量不能低于 95%。

本质上，吞吐量是指应用程序线程用时占程序总用时的比例。

例如，吞吐量99/100， 意味着100秒的程序执行时间，应用程序线程运行了99秒， 而在这一时间段内GC线程只运行了1秒。

### 核心指标2：停顿时间(pause times)：

JVM在专门的线程(GC threads)中执行GC。

因为JVM进行垃圾回收的时候，某些阶段必须要停止业务线程专心进行垃圾收集， 只要GC线程是活动的，它们将与应用程序线程(application threads)争用当前可用CPU的时钟周期。

停顿时间(pause times) 是指一个时间段内应用程序线程让与GC线程执行，而应用程序线程完全暂停。

例如，GC期间100毫秒的停顿时间， 意味着在这100毫秒期间内没有应用程序线程是活动的。

如果说一个正在运行的应用程序有100毫秒的“平均停顿时间”，那么就是说该应用程序所有的停顿时间平均长度为100毫秒。

同样，100毫秒的“最大停顿时间”是指：该应用程序所有的停顿时间最大不超过100毫秒。

注意，这里说的JVM停顿时间，就是指JVM停止业务线程而去进行垃圾收集的这段时长，其实指的是**每次GC造成用户线程停顿的平均时间**，不是总的垃圾回收时间。

停顿时间越长，就意味着GC场景下，用户线程平均等待的时间越长，停顿时间会直接影响用户使用系统的体验。

除了吞吐量(throughput) 、停顿时间(pause times) 两个核心指标，JVM调优还会关心下面的非核心指标：

### 核心指标3：堆内存占用量：

细致监控堆内存使用量、非堆内存使用量以及永久代（或元空间）使用量指标数据。

举例来说，当堆内存使用量持续增加，而内存回收频率较低时，可能暗示着潜在的内存泄漏问题，这可能导致系统性能下降或者最终的内存耗尽（OOM）。

### 非核指标1：垃圾回收次数

GC非常占用CPU资源的，如果GC占用的资源越多，那么意味着其他事情所用的资源会减少，系统所能做的事情也会越少。

尽管垃圾回收过程会消耗大量的CPU资源，但是我们也不能单纯地、一味的追求GC次数减少

为啥? GC次数减少了，有可能单次GC的时间变长，那么就可能会增加单次GC的“停顿时长”（核心指标2），

### 非核指标2：垃圾回收频率

通常情况下，与垃圾回收次数相比，较低的垃圾回收频率被认为是更好的选择。

垃圾回收的频率,需要适中

- 频率过小,每次垃圾回收的时间会过长
- 频率过大,停顿时间长,延迟高

所以：通常来说垃圾回收频率是越低越好。

详细记录GC频率、GC停顿时间以及每次GC后的内存情况。

或者说：减少 GC次数可能会导致单次垃圾回收的时间变长，进而增加单次垃圾回收的“停顿时长”。

所以， 需要在这两者之间做一些平衡。

## 吞吐量、暂停时间、堆内存占用三者之间的关系

这三个指标不可能同时达到,因为他们是一个不可能的关系

**内存变大,要回收的东西变多,暂停时间自然增加.**

**吞吐量增加,必然要降低垃圾回收频率,频率降低,垃圾谁收停顿时间必然增大.**

因此,目前gc的优化方向主要是**吞吐量和暂停时间.**

### “高吞吐量”和“低停顿时间”是一对相互竞争的目标

高吞吐量最好因为这会让应用程序的最终用户感觉只有应用程序线程在做“生产性”工作。

直觉上，吞吐量越高程序运行越快。

低停顿时间最好因为从最终用户的角度来看不管是GC还是其他原因导致一个应用被挂起始终是不好的。

这取决于应用程序的类型，有时候甚至短暂的200毫秒暂停都可能打断终端用户体验。

因此，具有低的最大停顿时间是非常重要的，特别是对于一个交互式应用程序。

不幸的是”高吞吐量”和”低停顿时间”是一对相互竞争的目标（矛盾）。

GC需要一定的前提条件以便安全地运行。

例如，必须保证应用程序线程在GC线程试图确定哪些对象仍然被引用和哪些没有被引用的时候不修改对象的状态。

为此，应用程序在GC期间必须停止(或者仅在GC的特定阶段，这取决于所使用的算法)。 然而这会增加额外的线程调度开销：直接开销是上下文切换，间接开销是因为缓存的影响。

加上JVM内部安全措施的开销，这意味着GC及随之而来的不可忽略的开销，将增加GC线程执行实际工作的时间。

因此我们可以通过尽可能少运行GC，来最大化吞吐量，例如，只有在不可避免的时候进行GC，来节省所有与它相关的开销。

然而，仅仅偶尔运行GC意味着每当GC运行时将有许多工作要做，因为在此期间积累在堆中的对象数量很高。 单个GC需要花更多时间来完成， 从而导致更高的平均和最大停顿时间。

因此，考虑到低停顿时间，最好频繁地运行GC以便更快速地完成。这反过来又增加了开销并导致吞吐量下降，我们又回到了起点。

综上所述，在设计（或使用）GC算法时，我们必须确定我们的目标：

一个GC算法只可能针对两个目标之一（即只专注于最大吞吐量或最小停顿时间），或尝试找到一个二者的折衷。

### 吞吐量和暂停时间是矛盾的,如何抉择?

高吞吐量较好因为这会让应用程序的最终用户感觉只有应用程序线程在做"生产性"工作。直觉上,吞吐量越高程序运行越快。

低暂停时间(低延迟)较好因为从最终用户的角度来看不管是GC还是其他原因导致一个应用被挂起始终是不好的。这取决于应用程序的类型, 有时候甚至短暂的200毫秒暂停都可能打断终端用户体验 。因此,具有低的较大暂停时间是非常重要的,特别是对于一个 交互式应用程序 。

不幸的是"高吞吐量"和"低暂停时间"是一对相互竞争的目标(矛盾)。

- 因为如果选择以吞吐量优先,那么 必然需要降低内存回收的执行频率 ,但是这样会导致GC需要更长的暂停时间来执行内存回收。
- 相反的,如果选择以低延迟优先为原则,那么为了降低每次执行的内存回收时的暂停时间,也 只能频繁地执行内存回收 ,但这又引起了年轻代内存的缩减和导致程序吞吐量的下降。

在设计(或使用)GC算法时,我们必须确定我们的目标: 一个GC算法可能针对两个目标之一(即只专注于较大吞吐量或最小暂停时间),或尝试找到一个二者的折中。

现在标准: 在最大吞吐量优先的情况下,降低停顿时间

### 不同的垃圾回收器有不同的抉择方向:

- Parallel以吞吐量优先
- cms以停顿时间优先
- 而G1则取折中方案: 在保证用户可接受的停顿时间的前提下,尽可能提高吞吐量.

JVM调优没有万能的公式和标准，因为每个人所面对的场景是不一样。

要想调整到最优的性能，其实首先要确认的是自己的需求目标是什么(以吞吐量优先/停顿时间优先)，

然后，根据这个目标去慢慢的调整各项指标，从而达到一个最佳的平衡点。

## 三：如果获得JVM内存指标?

在项目启动的时候 增加下列参数来收集GC日志，然后通过第三方的日志分析工具（比如GCesay:https://gceasy.io/）

分析收集到的GC日志来得到吞吐量、停顿时间相关的统计数据。

```
 java  
 -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
 -XX:+UseGCLogFileRotation 
 -XX:+PrintHeapAtGC -XX:NumberOfGCLogFiles=5  
 -XX:GCLogFileSize=20M    
 -Xloggc:/opt/ard-user-gc-%t.log  
 -jar abg-user-1.0-SNAPSHOT.jar 
 -Xloggc:/opt/app/ard-user/ard-user-gc-%t.log   设置日志目录和日志名称
 -XX:+UseGCLogFileRotation           开启滚动生成日志
 -XX:NumberOfGCLogFiles=5            滚动GC日志文件数，默认0，不滚动
 -XX:GCLogFileSize=20M               GC文件滚动大小，需开启UseGCLogFileRotation
 -XX:+PrintGCDetails                 开启记录GC日志详细信息（包括GC类型、各个操作使用的时间）,并且在程序运行结束打印出JVM的内存占用情况
 -XX:+ PrintGCDateStamps             记录系统的GC时间           
 -XX:+PrintGCCause                   产生GC的原因(默认开启)
```

### 日志分析工具有哪些？

我们看到日志，尤其是CMS和G1的日志，直接看日志文档都是很不方便的，密密麻麻的文字，其实市面上已经有一些日志分析工具了。

进行系统调优时，首先需要对系统的各项指标进行检测。为了有效地进行监测和设置相应的阈值，我们通常会借助监控工具，例如普罗米修斯等。在分析阶段，以下工具是常用的：

1. VisualVM：

   这是一个功能强大、可扩展的开源工具，用于深入分析Java应用程序。它提供了丰富的功能，包括性能监控、内存使用情况、垃圾回收情况等，并支持线程分析、堆快照等功能。

2. Java Mission Control (JMC)：

   由Oracle提供的专业Java性能监控和故障诊断工具。JMC集成了多种强大功能，包括垃圾回收分析、内存泄漏检测、线程分析等。

3. jvisualvm：

   这是JDK自带的监控和调试工具，可用于监视本地和远程Java应用程序的性能、内存使用情况等。它提供了直观的图形界面和丰富的监控指标。

4. JConsole：

   JConsole是JDK自带的监控工具，提供了基本的图形界面，可用于监视Java应用程序的内存使用情况、线程信息、垃圾回收情况等。

5. GCViewer：

   这是专门用于分析Java应用程序垃圾回收日志的工具。

   GCViewer能将GC日志解析成易于理解的图表和统计信息，帮助用户分析和优化垃圾回收行为。

接下来，介绍使用 gceasy.io 进行 日志分析

网址：https://gceasy.io/

注意：这款工具不需要我们下载软件，他是在线的。

我们要做的就是两步：

步骤一：导出GC日志到本地磁盘

步骤二：将本地日志上传到gceasy.io上，进行分析

### 指标分析第一步：导出日志

```
-Xloggc:/Users/lxl/Downloads/gc.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-XX:+PrintGCCause
```

- ‐Xloggc参数：指定gc日志的保存地址。这里指定的是当前目录，文件名以gc-+时间戳.log打印。%t表示时间戳
- ‐XX:+PrintGCDetails：在日志中打印GC详情。
- ‐XX:+PrintGCDateStamps：在日志中打印GC的时间
- ‐XX:+PrintGCTimeStamps：在日志中打印GC耗时
- ‐XX:+PrintGCCause ： [这个参数没查到]
- ‐XX:+UseGCLogFileRotation：这个参数表示以滚动文件的形式打印日志
- ‐XX:NumberOfGCLogFiles：GC日志文件的最大个数，这里设置10个
- ‐XX:GCLogFileSize：GC日志每个文件的最大容量，这里是100M

我们把日志下载到Downloads文件夹下了。以下便是GC日志的全部内容

```
Java HotSpot(TM) 64-Bit Server VM (25.202-b08) for bsd-amd64 JRE (1.8.0_202-b08), built on Dec 15 2018 20:16:16 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 16777216k(1745536k free)
/proc/meminfo:
CommandLine flags: -XX:-BytecodeVerificationLocal -XX:-BytecodeVerificationRemote -XX:InitialHeapSize=268435456 -XX:+ManagementServer -XX:MaxHeapSize=4294967296 -XX:+PrintGC -XX:+PrintGCCause -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:TieredStopAtLevel=1 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
2022-01-12T15:02:37.044-0800: 0.839: [GC (Allocation Failure) [PSYoungGen: 65536K->4400K(76288K)] 65536K->4416K(251392K), 0.0043915 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2022-01-12T15:02:37.308-0800: 1.103: [GC (Allocation Failure) [PSYoungGen: 69936K->4959K(76288K)] 69952K->5047K(251392K), 0.0046449 secs] [Times: user=0.02 sys=0.01, real=0.01 secs] 
2022-01-12T15:02:37.625-0800: 1.420: [GC (Allocation Failure) [PSYoungGen: 70495K->7467K(76288K)] 70583K->7563K(251392K), 0.0051392 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
2022-01-12T15:02:37.831-0800: 1.627: [GC (Allocation Failure) [PSYoungGen: 73003K->9356K(141824K)] 73099K->9460K(316928K), 0.0072596 secs] [Times: user=0.03 sys=0.01, real=0.00 secs] 
2022-01-12T15:02:37.869-0800: 1.664: [GC (Metadata GC Threshold) [PSYoungGen: 22322K->7049K(141824K)] 22426K->7161K(316928K), 0.0057809 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
2022-01-12T15:02:37.875-0800: 1.670: [Full GC (Metadata GC Threshold) [PSYoungGen: 7049K->0K(141824K)] [ParOldGen: 112K->6873K(87040K)] 7161K->6873K(228864K), [Metaspace: 20573K->20571K(1067008K)], 0.0237404 secs] [Times: user=0.09 sys=0.01, real=0.02 secs] 
2022-01-12T15:02:38.392-0800: 2.188: [GC (Allocation Failure) [PSYoungGen: 131072K->7194K(236032K)] 137945K->14075K(323072K), 0.0054542 secs] [Times: user=0.01 sys=0.01, real=0.00 secs] 
2022-01-12T15:02:39.850-0800: 3.646: [GC (Allocation Failure) [PSYoungGen: 235546K->9697K(270336K)] 242427K->20203K(357376K), 0.0092838 secs] [Times: user=0.02 sys=0.01, real=0.01 secs] 
2022-01-12T15:02:40.479-0800: 4.274: [GC (Metadata GC Threshold) [PSYoungGen: 179780K->12779K(397312K)] 190286K->25839K(484352K), 0.0117953 secs] [Times: user=0.04 sys=0.01, real=0.02 secs] 
2022-01-12T15:02:40.491-0800: 4.286: [Full GC (Metadata GC Threshold) [PSYoungGen: 12779K->0K(397312K)] [ParOldGen: 13059K->21448K(132096K)] 25839K->21448K(529408K), [Metaspace: 34068K->34068K(1079296K)], 0.0437361 secs] [Times: user=0.16 sys=0.01, real=0.04 secs] 
2022-01-12T15:02:42.177-0800: 5.972: [GC (Allocation Failure) [PSYoungGen: 384512K->13185K(399872K)] 405960K->34641K(531968K), 0.0115070 secs] [Times: user=0.04 sys=0.01, real=0.01 secs] 
2022-01-12T15:02:43.010-0800: 6.806: [GC (Allocation Failure) [PSYoungGen: 397697K->16864K(530432K)] 419153K->58461K(662528K), 0.0248406 secs] [Times: user=0.04 sys=0.02, real=0.02 secs] 
2022-01-12T15:02:44.338-0800: 8.133: [GC (Allocation Failure) [PSYoungGen: 530400K->26083K(539648K)] 571997K->86488K(671744K), 0.0302789 secs] [Times: user=0.06 sys=0.02, real=0.03 secs] 
2022-01-12T15:02:45.800-0800: 9.595: [GC (Allocation Failure) [PSYoungGen: 539619K->32647K(733696K)] 600024K->99769K(865792K), 0.0280332 secs] [Times: user=0.04 sys=0.02, real=0.02 secs] 
2022-01-12T15:02:47.765-0800: 11.560: [GC (Allocation Failure) [PSYoungGen: 729479K->41445K(738304K)] 796601K->124936K(870400K), 0.0370655 secs] [Times: user=0.04 sys=0.02, real=0.04 secs] 
2022-01-12T15:02:49.620-0800: 13.415: [GC (Allocation Failure) [PSYoungGen: 738277K->26677K(974848K)] 821768K->114930K(1106944K), 0.0270382 secs] [Times: user=0.05 sys=0.02, real=0.02 secs] 
2022-01-12T15:02:52.146-0800: 15.942: [GC (Allocation Failure) [PSYoungGen: 959541K->17569K(985600K)] 1047794K->110447K(1117696K), 0.0274985 secs] [Times: user=0.05 sys=0.01, real=0.03 secs] 
2022-01-12T15:02:54.110-0800: 17.905: [GC (Allocation Failure) [PSYoungGen: 950433K->10240K(1236480K)] 1043311K->109662K(1368576K), 0.0146713 secs] [Times: user=0.05 sys=0.01, real=0.01 secs] 
2022-01-12T15:02:54.692-0800: 18.487: [GC (Metadata GC Threshold) [PSYoungGen: 264005K->3360K(1259520K)] 363427K->109573K(1391616K), 0.0086901 secs] [Times: user=0.03 sys=0.01, real=0.01 secs] 
2022-01-12T15:02:54.701-0800: 18.496: [Full GC (Metadata GC Threshold) [PSYoungGen: 3360K->0K(1259520K)] [ParOldGen: 106213K->54092K(208384K)] 109573K->54092K(1467904K), [Metaspace: 56204K->56204K(1101824K)], 0.1487173 secs] [Times: user=0.69 sys=0.01, real=0.14 secs] 
2022-01-12T15:02:57.787-0800: 21.583: [GC (Allocation Failure) [PSYoungGen: 1209856K->49146K(1321984K)] 1263948K->116260K(1530368K), 0.0339265 secs] [Times: user=0.05 sys=0.01, real=0.04 secs] 
2022-01-12T15:03:16.198-0800: 39.994: [GC (Allocation Failure) [PSYoungGen: 1321978K->29589K(1335296K)] 1389092K->101049K(1543680K), 0.0214759 secs] [Times: user=0.06 sys=0.01, real=0.03 secs] 
2022-01-12T15:03:19.021-0800: 42.816: [GC (GCLocker Initiated GC) [PSYoungGen: 1302421K->60915K(1280512K)] 1373881K->180735K(1488896K), 0.0482886 secs] [Times: user=0.08 sys=0.01, real=0.05 secs] 
2022-01-12T15:03:21.847-0800: 45.642: [GC (Allocation Failure) [PSYoungGen: 1280499K->89087K(1308672K)] 1400321K->228379K(1517056K), 0.0336500 secs] [Times: user=0.10 sys=0.01, real=0.04 secs] 
2022-01-12T15:03:24.516-0800: 48.311: [GC (Allocation Failure) [PSYoungGen: 1308671K->67295K(1257472K)] 1447963K->225652K(1465856K), 0.0381420 secs] [Times: user=0.07 sys=0.02, real=0.04 secs]
```

### 指标分析第二步：导入分析工具，尽心分析

打开gceasy.io网站，并选择本地的gc文件，然后点击分析。

（分析的速度根据日志的多少而定，可能会比较慢）

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6V9atGYuUic6CY98FyNAUArIibBJPsYo8jXdqIJsbBhMCwxLHwCdkeVIRg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

接下来看看分析结果：

### JVM memory size (JVM内存大小)

GCEasy是一款非常好用的在线分析GC日志的工具，打开官网，直接上传gc日志，也可以更加上门的要求进行压缩上传。![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VxbfNE4Cj5PgNcCO6rdtSicLYoehB2fdpV8VPuCNzlBjxJmePuicqaJXg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里的Allocated和Peak分别表示可分配空间和峰值

- **Allocated**：可分配空间大小。

  **具体含义如下**：指示为每一代分配的大小。此数据点是从GC日志收集的，因此它可能与JVM系统属性指定的大小相匹配，也可能不匹配。假设您已将总堆大小配置为2gb，而在运行时，如果JVM只分配了1gb，那么在本报告中，您将看到分配的大小仅为1gb

- **Peak：** 分配的峰值。

  **具体含义如下**：每一代的峰值内存利用率。通常它不会超过分配的大小。然而，在少数情况下，我们也看到峰值利用率超出了分配的大小，特别是在G1 GC中

JVM memory size ，GCEasy展示了年轻代、老年代、元空间。JVM给分配的大小和程序运行过程中使用的峰值大小。

从JVM memory size展示的信息，我们可以判断是否需要做下面的几件事情。

- 是否需要修改JVM内存（-Xms、-Xmx、-Xmn…）相关配置，比如年轻代和老年代峰值远远小于分配的大小，这个时候我们可以适当的减小内存设置。
- 是否需要调整年轻代和老年代的比例(-XX:NewSize(-Xns)、-XX:MaxNewSize(-Xmn)、-XX:SurvivorRatio=8)。比如老年大的峰值一直小于老年代申请的内存，这个时候我们可以稍微多分点空间给年轻代。
- 是否需要修改元空间(XX:MetaspaceSize，-XX:MaxMetaspaceSize)相关设置。 **年轻代，老年代属于堆区，元空间属于非堆区（直接对接的是机器的内存）**

### Key Performance Indicatiors（关键指标）

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VNGZKJCRuN84jlGOibc8XORuBiboQp9GV6nBUYZZcu2NMfutibTNxk99fA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- Throughput：吞吐量。

  指的是处理实际事务花费的时间与GC花费的时间的百分比。这个值越高越好

- Latency：

  延迟情况。这里的延迟情况是指的GC过程花费的时间。具体含义如上图

Throughput表示的是吞吐量 Latency表示响应时间 Avg Pause GC Time 平均GC时间 Max Pause GC TIme 最大GC时间

Key Performance Indicators 给我们展示了GC吞吐量（应用程序线程用时占程序总用时的比例，越高越好），每次GC的平均耗时（建议控制在50ms以下），GC最长耗时，每个时间段的GC次数及占比信息。

通过Key Performance Indicators显示的信息里面，我们需要关注下面几个问题：

- 吞吐量，应用花在非GC上的时间百分比（引用花在生产任务上的百分比）。所以吞吐量越高越好。
- 每次GC的平均耗时。越小越好，建议50ms以下。
- GC最长耗时。越小越好。如果你的应用是一个后台程序，并且任何请求不超过10秒，那么GC最长耗时就不能超过10秒。

### Interactive Graphs(交互图)

Interactive Graphs 展示了

**Heap after GC**：GC之后堆的使用情况 **Heap before GC**：GC之前堆的使用情况 **GC Duration**：GC持续时间 **Reclaimed Bytes**：GC回收掉的垃圾对象的内存大小 **Young Gen**：年轻代堆的使用情况 **Old Gen**：老年代堆的使用情况 **Meta Space**：元空间的使用情况 **A & P**：每次GC的时候堆内存分配和晋升情况。其中红色的线表示每次GC的时候年轻代里面有多少内存(对象)晋升到了老年代。

第一部分是Heap after GC，GC后堆的内存图，堆是用来存储对象的，从图中可以看出，随着GC的进行，垃圾回收器把对象都回收掉了，因此堆的大小逐渐增大。![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6V2SicRqEcZ94xGDKHV2PicF7bHp4nq9wu8AnqSYpFIWoegYkaGS27SGZw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)第二部分是Heap before GC，这是GC前堆的使用率，可以看出随着程序的运行，堆使用率越来越高，堆被对象占用的内存越来越大。![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VNMX1sOVgpKIib8Suvp6eS1Y4RsWVF7tFCLOKMn6hLRs2IFUiamibpvyFA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)第三部分是GC Duration Time，就是GC持续时间。一个GC事件的发生具有多个阶段，而不同的垃圾回收器又有不同的阶段，这里展示不作细分。这些阶段（例如并发标记，并发清除等）与程序线程一起并发运行，此时不会暂停程序线程。但是某些阶段（例如初始标记，清除等）会暂停整个应用程序，所以此图标描述的仅暂停阶段所花费的时间。![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VYPT2pjjYIpUCXQOanzDnrGQeH4sUpG0Vd1WktDxurxvggAvCxh2kicQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)第四部分表示的是GC回收掉的垃圾对象的内存大小。![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VFAaSPnN7P34164wape9B0odScDSicDDPCunD6AgfSux8rEnkIibTkU9g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)第五部分表示的是Young Gen，年轻代的内存分配情况。对象都是朝生夕死，年轻代存放的就是刚刚产生的对象，每进行一次GC，都会GC掉很多垃圾对象，剩下的就是右GC Root关联的对象，这些对象会年龄会逐渐增加，达到了一定阈值就会晋升为老年代的对象。可以看到before GC表示的图线随着时间的进行逐渐增大，也就是年轻代中对象越来越多，而GC事件发生后，年轻代中对象就会减少，也就是after GC图线表示的内存变化趋势。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VJbXyXsiaI3eyICSoS9Aics0tHkkNP7k3ZuzxHUGpKtFwCPv8GJy0u5gQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)第六部分是Old Gen，表示的是老年代的内存分配情况。细心的读者会发现，为啥一开始before GC的内存大小比after GC的内存分配要少呢？这里得先知道老年代存放的都是年龄大的对象，意思就是经过了多次GC都没有被GC掉的对象，就会晋升为老年代的对象。所以这就解释了为啥after GC内存要比before GC内存要大，因为每次GC过后，都会有年轻代的对象晋升为老年代对象。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VobhwVD1bT4bib1Uofic0coBQpvqRIWqkTicLIUHPRG5jBiaDEZaLh01OUg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)第七部分是每次GC的时候堆内存分配和晋升情况。其中红色的线表示每次GC的时候年轻代里面有多少内存(对象)晋升到了老年代。![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VhwvmbQnxPDUypmbWaQaGHoHOAKNVn2VXsvVuSLymgpXF4PiabQyjkgA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### GC Statistics(GC统计信息)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6V9gBhGoHsUibheRqiaub7ONsXHvCmv0gjQIMEPeebrrsENM7OdSQo5E4g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)GC Statistics显示一些GC的统计信息。

每种GC总共回收了多少内存、总共用了多长时间、平均时间、以及每种GC的单独统计信息啥的。

### Object Stats(对象的一些统计信息)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VlXibUfJQ3RcFs2dnxDbz6h5MeRgwibZvU1r8FuWQ5ZAZJ80ga0ibL4hlw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### GC Causes(GC的原因信息)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6V09eic7s2SMvEPj8R27ic1FibIY0UiaM9XnW1dnPaf3NJiaEicoYoIBI7HuqQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### Memory Leak

由于记录的程序没有内存泄漏，所以这里就没有内存泄漏的日志信息。

此处可以诊断8种OOM中的5种（Java堆内存溢出，超出GC开销限制，请求数组大小超过JVM限制，Permgen空间，元空间）。

## 四：JVM 常用配置策略

### 垃圾回收器的选择

选择垃圾回收器时，应根据CPU核心数、关注点（吞吐量或用户停顿时间）以及JDK版本等因素做出合适的选择，以提高应用程序的性能和稳定性。

- **CPU单核：**

  当系统仅有单核CPU时，Serial垃圾收集器是最佳选择。

  由于单核系统的性能瓶颈主要集中在单一处理器上，使用Serial垃圾收集器能够简化垃圾回收的过程，提高系统的整体性能。

- **CPU多核：关注吞吐量**

  对于多核CPU且关注系统吞吐量的情况，推荐选择Parallel Scavenge（PS）加 Parallel Old（PO）的组合。

  这种组合利用了多核CPU的并行处理能力，通过并行处理新生代和老年代的垃圾收集，以提高系统的吞吐量和整体性能。

- **CPU多核，关注用户停顿时间，JDK版本1.6或1.7：**

  如果系统是多核CPU，并且更关注用户停顿时间，特别是在JDK版本为1.6或1.7的情况下，推荐选择Concurrent Mark-Sweep（CMS）垃圾收集器。

  CMS垃圾收集器以减少应用程序停顿时间为目标，通过与应用程序线程并发执行部分垃圾回收操作，从而降低了GC造成的停顿时间，提高了系统的响应速度和用户体验。

- **CPU多核，关注用户停顿时间，JDK1.8及以上，JVM可用内存6G以上：**

  对于JDK版本为1.8及以上，并且系统具备充足的内存资源（6G及以上），且依然关注用户停顿时间的情况，推荐选择Garbage-First（G1）垃圾收集器。

  G1垃圾收集器是一种面向服务端应用的垃圾收集器，具有高效的垃圾回收、可预测的停顿时间和良好的内存整理能力，适用于对用户停顿时间有较高要求的应用场景。

**垃圾回收器的选择 的切换配置：**

```
//设置Serial垃圾收集器（新生代）
 开启：-XX:+UseSerialGC
 
 //设置PS+PO,新生代使用功能Parallel Scavenge 老年代将会使用Parallel Old收集器
 开启 -XX:+UseParallelOldGC
 
 //CMS垃圾收集器（老年代）
 开启 -XX:+UseConcMarkSweepGC
 
 //设置G1垃圾收集器
 开启 -XX:+UseG1GC
```

### JVM参数常用原则

- 对于JVM堆的设置

  通常我们会使用 **-Xms** 和 **-Xmx** 来设定最小和最大堆大小，将它们设置为相同的值可以防止垃圾收集器在堆大小之间进行收缩，从而减少额外的时间消耗。

- 年轻代和年老代的大小将根据默认比例（通常为1：2）分配堆内存。

  我们可以通过调整 **-XX:NewRatio** 参数来调整它们之间的比例，

  也可以通过 **-XX:NewSize** 和 **-XX:MaxNewSize** 来设置年轻代的绝对大小。

  为了防止年轻代堆大小的调整，通常将 **-XX:NewSize** 和 **-XX:MaxNewSize** 设置为相同大小。

- 年轻代和年老代大小的合理设置没有标准答案，因此调优时需要观察它们大小变化对系统的影响。

  更大的年轻代会延长普通GC周期但增加每次GC的时间，而更小的年老代会导致更频繁的Full GC。

  选择应根据应用程序对象生命周期的分布情况，例如，如果应用存在大量的临时对象，则应选择更大的年轻代；如果存在大量的持久对象，则应适当增大年老代。

  观察应用一段时间后，根据峰值时年老代所占内存来调整年轻代的大小，但应保留年老代至少1/3的增长空间。

- 在配置较好的机器上（如多核、大内存），可以为年老代选择并行收集算法，使用**XX:+UseParallelOldGC**，默认为串行收集。

- 线程堆栈的设置：每个线程默认会分配1M的堆栈空间，用于存放栈帧、调用参数、局部变量等。

  对于大多数应用来说，这个默认值过大，一般可以将其减小至256K。

  减小线程堆栈大小可以在内存不变的情况下创建更多线程，但这也受限于操作系统的支持。

## 五：常见调优策略

### 5.1 调整内存大小

- 现象：垃圾收集频率非常频繁
- 措施：考虑增加堆内存大小。
- 说明：

频繁的垃圾收集通常是由于内存过小，导致需要不断进行垃圾收集以释放空间来容纳新对象。

因此，增加堆内存大小可以显著降低垃圾收集的频率。

需要注意的是，如果垃圾收集次数虽然频繁但每次回收的对象却很少，那么问题可能不在于内存过小，而是由于内存泄漏导致的对象无法被正确回收，从而引发了频繁的垃圾收集。

在这种情况下，调整内存大小可能无法解决问题，而需要对代码进行进一步的分析和调试。

```
 //设置堆初始值
 指令1：-Xms2g
 指令2：-XX:InitialHeapSize=2048m
 
 //设置堆区最大值
 指令1：`-Xmx2g` 
 指令2： -XX:MaxHeapSize=2048m
 
 //新生代内存配置
 指令1：-Xmn512m
 指令2：-XX:MaxNewSize=512m
```

### 5.2 调整GC触发时机

- 现象：

  在CMS和G1垃圾回收器下，频繁发生Full GC，导致程序严重卡顿。

- 说明：

在G1和CMS的部分GC阶段是并发进行的，即业务线程和垃圾收集线程同时运行。

这意味着在垃圾收集过程中，业务线程可能会生成新的对象。

因此，在进行垃圾收集时，需要预留一部分内存空间来容纳新产生的对象。

如果此时内存空间不足以容纳新对象，JVM会停止并发收集，暂停所有业务线程（STW），以确保垃圾收集正常进行。

可以通过调整GC的触发时机（例如在老年代占用60%时触发GC）来预留足够的空间给业务线程创建的对象。

```
 //使用多少比例的老年代后开始CMS收集，默认是68%，如果频繁发生SerialOld卡顿，应该调小
 -XX:CMSInitiatingOccupancyFraction
 
 //G1混合垃圾回收周期中要包括的旧区域设置占用率阈值。默认占用率为 65%
 -XX:G1MixedGCLiveThresholdPercent=65 
```

### 5.3 调整对象晋升到老年代年龄阈值

- 现象：

  老年代发生频繁的GC，每次清理回收大量对象。

- 说明：

当对象的晋升年龄设定较低时，新生代中的对象很快就会被晋升到老年代。

这导致老年代中对象数量增多，其中很多对象实际上在短时间内就可能被回收。

通过调整对象的晋升年龄，可以减少过早进入老年代的对象数量，从而减少老年代的空间压力和频繁的GC。

注意：提高晋升年龄虽然可以减缓老年代的压力，但同时可能会增加新生代的GC频率，因为对象在新生代的停留时间变长。

此外，新生代中频繁复制这些对象可能会导致新生代的GC时间也相应增长。

在调整晋升年龄时，应综合考虑新生代和老年代的GC性能，以达到最优的系统性能平衡。

```
 // 进入老年代最小的GC年龄,年轻代对象转换为老年代对象最小年龄值，默认值7
 -XX:InitialTenuringThreshol=7 
```

### 5.4 调整大对象进入老年代的标准

- 现象：

  老年代经常发生频繁的GC，每次回收大量对象，而这些对象的体积都相对较大。

- 说明：

大量大对象直接分配到老年代会快速填满老年代空间，导致老年代频繁GC。

为解决此问题，可调整大对象直接进入老年代的标准。

需要注意：将大对象调整为直接进入老年代后，可能会增加新生代的GC频率和时间。

```
 //新生代可容纳的最大对象,大于则直接会分配到老年代，0代表没有限制。
  -XX:PretenureSizeThreshold=1000000 
```

### 5.5 调整内存区域大小比率

- 现象：

  某一内存区域频繁发生GC，而其他区域的GC表现正常。

- 说明：

频繁的GC可能是由于对应区域的空间不足所致，需要不断进行GC以释放空间。

在JVM堆内存无法增加的情况下，可以考虑调整对应区域的大小比率。

注意：尽管频繁的GC可能是由于空间不足造成的，但也有可能是因为内存泄漏导致内存无法回收，进而引发GC频繁。

因此，在调整内存区域大小比率之前，需要仔细分析是否存在内存泄漏问题。

```
 // survivor区和Eden区大小比率
 指令：-XX:SurvivorRatio=6  //S区和Eden区占新生代比率为1:6,两个S区2:6
 
 // 新生代和老年代的占比
 -XX:NewRatio=4  //表示新生代:老年代 = 1:4 即老年代占整个堆的4/5；默认值=2
```

### 5.6 调整对象晋升至老年代的年龄阈值

- 现象：

  老年代频繁进行GC，每次回收大量对象。

- 说明：

如果对象的晋升年龄较小，新生代中的对象很快就会晋升至老年代，导致老年代中对象数量增多。

然而，这些对象在接下来的短时间内可能会被回收。为解决老年代空间不足导致的频繁GC问题，可调整对象晋升至老年代的年龄阈值，使对象不那么容易晋升至老年代。

注意：增加对象晋升年龄可能会导致新生代中对象的停留时间增加，从而增加新生代的GC频率，并且复制大对象可能导致新生代GC的时间延长。

在调整晋升年龄时，需综合考虑新生代和老年代的GC性能，以获得最优的系统性能平衡。

### 5.7 调整垃圾回收的触发时机

- 现象：

  G1和CMS垃圾收集器在执行垃圾回收时与应用程序的业务线程并发工作。

  在垃圾回收过程中，业务线程可能生成新对象，需预留内存空间以容纳这些新产生的对象。

  若内存空间不足，JVM会暂停所有业务线程（STW）以确保垃圾回收正常进行。

- 说明：

在进行垃圾回收时，若未预留足够的内存空间供新对象使用，可能导致内存压力过大，从而触发STW。

通过调整垃圾回收的触发时机来预留足够的内存空间，如可设定在老年代占用达到一定比例时触发垃圾回收。

这有助于提前释放内存空间，为新对象分配留出足够的空间，从而减少因内存不足而导致的STW情况。

注意：

提早触发垃圾回收会增加老年代垃圾回收的频率，这可能导致一些性能开销，如额外的CPU使用和系统停顿时间。

因此，在调整垃圾回收的触发时机时，需要在性能与内存利用率之间找到恰当的平衡。

### 5.8 设置符合预期的停顿时间

> **现象**： 程序间接性的卡顿 **原因**：如果没有确切的停顿时间设定，垃圾收集器以吞吐量为主，那么垃圾收集时间就会不稳定。 **注意**：不要设置不切实际的停顿时间，单次时间越短也意味着需要更多的GC次数才能回收完原有数量的垃圾.

参数配置：l

```
 //GC停顿时间，垃圾收集器会尝试用各种手段达到这个时间
      -XX:MaxGCPauseMillis 
```

## 六： JVM调优案例和实践

### 案例1：网站流量增加后，网页响应速度变慢

**问题描述** 在测试环境中，网站速度较快，但一到生产环境就显著变慢。 **问题分析**

1. **初步诊断**： 通过使用 **jstat -gc** 指令监控线上JVM的GC活动，发现GC频率和所占时间异常高。这表明频繁的GC正影响业务线程的执行，从而导致页面响应缓慢。
2. **内存调整后的问题**： 增加JVM的堆内存从2GB到16GB后，虽然常规请求的处理速度提高，但出现了间歇性的更长时间卡顿。进一步监控发现，虽然Full GC（FGC）的次数不多，但每次的持续时间过长，有时达到几十秒。
3. **原因推断**： 增加堆内存后，虽然减少了频繁的垃圾回收，但因为PS+PO垃圾收集器（Parallel Scavenge + Parallel Old）在垃圾标记和收集阶段都需要停止所有工作线程（STW），所以每次GC时业务线程的停顿时间显著增长。

**解决方案**

1. **调整垃圾收集器**： 服务不稳定的根本问题是垃圾回收过程中的停顿时间过长，由于默认的PS+PO组合垃圾收集器导致。为了解决这一问题，可更换为并发类的收集器，如CMS垃圾收集器。
2. **CMS配置优化**： 根据系统运行的实际情况，调整CMS的启动阈值，预设了合理的停顿时间，以确保不会因为内存回收而影响用户的使用体验。

### 案例2：CPU飙升和GC频繁的调优实践

**问题描述：** 随着在线游戏玩家数量的增加，系统出现CPU飙升和GC频繁的情况，导致游戏体验下降。 **问题分析：** 使用监控工具检查系统的CPU使用情况和GC情况，发现系统在高负载情况下CPU占用过高，且GC频率过于频繁。 **解决方案：**

1. **代码优化**：进行代码审查和性能分析，发现并优化存在不必要的循环操作和资源竞争问题，以减少CPU占用。
2. **堆内存调整**：增加堆内存大小，减少GC的频率，提高系统的吞吐量和稳定性，确保系统能够应对增加的玩家数量。
3. **GC算法调优**：根据系统负载情况和硬件环境，选择合适的GC算法，并调整相应的参数，以减少GC造成的性能损耗。例如，针对大堆内存和高并发情况，可以考虑使用并行GC或G1收集器，并根据具体情况调整相关参数以提升性能。

### 案例3：数据分析平台系统频繁 Full GC

**问题描述：** 数据分析平台对用户在App中的行为进行定时分析统计，但系统频繁发生Full GC，导致页面打开卡顿，影响用户体验。 **问题分析：**

1. **CMS GC算法使用**：系统使用CMS（Concurrent Mark-Sweep）GC算法，但频繁的Full GC表明GC调优方面存在问题。
2. **Young GC后存活对象进入老年代**：使用jstat命令监控发现，每次Young GC后大约有10%的存活对象进入老年代，这意味着Survivor区空间可能设置过小，导致存活对象在Survivor区放不下而提前进入老年代。

**解决方案：**

1. **调整Survivor区大小**：增大Survivor区大小，确保其能容纳Young GC后的存活对象，使存活对象能在Survivor区经历多次Young GC达到年龄阈值后才进入老年代。
2. **优化存活对象进入老年代的大小**：调整Survivor区大小后，每次Young GC后进入老年代的存活对象稳定在几百KB左右，大大降低了Full GC的频率，提升了系统的稳定性和性能。

### 案例4：内存飙高问题定位

**问题描述**：在Java进程中，内存飙高，可能是由于大量对象创建或内存泄漏导致的。

持续的内存飙高可能表明垃圾回收跟不上对象创建速度，或存在内存泄漏导致对象无法回收。 **问题分析：**

1. **观察垃圾回收情况：**
   - 使用 **jstat -gc PID 1000** 命令观察GC次数、时间等信息，每隔一秒打印一次。
   - 使用 **jmap -histo PID | head -20** 命令查看堆内存占用空间最大的前20个对象类型。
   - 如果GC频率高且每次回收的内存空间正常，可能是对象创建速度过快导致内存占用高；如果每次回收的内存很少，可能是内存泄漏。
2. **导出堆内存文件快照：**
   - 使用 **jmap -dump:live,format=b,file=/home/myheapdump.hprof PID** 命令将堆内存信息导出到文件，以进一步分析内存占用情况。

**解决方案：** 通过使用VisualVM对dump文件进行离线分析，识别内存占用较高的对象，并进一步定位到创建这些对象的业务代码位置，以便从代码和业务场景中精确定位具体问题。

### 案例5：Major GC和Minor GC频繁

这个案例，来自美团技术官网

#### 确定目标

服务情况：Minor GC每分钟100次 ，Major GC每4分钟一次，单次Minor GC耗时25ms，单次Major GC耗时200ms，接口响应时间50ms。

由于这个服务要求低延时高可用，结合上文中提到的GC对服务响应时间的影响，计算可知由于Minor GC的发生，12.5%的请求响应时间会增加，其中8.3%的请求响应时间会增加25ms，可见当前GC情况对响应时间影响较大。

*（50ms+25ms）× 100次/60000ms = 12.5%，50ms × 100次/60000ms = 8.3%* 。

优化目标：降低TP99、TP90时间。

#### 优化

首先优化Minor GC频繁问题。通常情况下，由于新生代空间较小，Eden区很快被填满，就会导致频繁Minor GC，因此可以通过增大新生代空间来降低Minor GC的频率。例如在相同的内存分配率的前提下，新生代中的Eden区增加一倍，Minor GC的次数就会减少一半。

这时很多人有这样的疑问，扩容Eden区虽然可以减少Minor GC的次数，但会增加单次Minor GC时间么？根据上面公式，如果单次Minor GC时间也增加，很难保证最后的优化效果。

结合下面情况来分析，单次Minor GC时间主要受哪些因素影响？是否和新生代大小存在线性关系？

首先，单次Minor GC时间由以下两部分组成：T1（扫描新生代）和 T2（复制存活对象到Survivor区）如下图。（注：这里为了简化问题，我们认为T1只扫描新生代判断对象是否存活的时间，其实该阶段还需要扫描部分老年代，后面案例中有详细描述。）

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6V8vDX3rXnpKWS6DibuWibEIibmu7JCDNG1zBluVfdk6RlVY9icoyyicFhEcQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 扩容前：新生代容量为R ，假设对象A的存活时间为750ms，Minor GC间隔500ms，那么本次Minor GC时间= T1（扫描新生代R）+T2（复制对象A到S）。
- 扩容后：新生代容量为2R ，对象A的生命周期为750ms，那么Minor GC间隔增加为1000ms，此时Minor GC对象A已不再存活，不需要把它复制到Survivor区，那么本次GC时间 = 2 × T1（扫描新生代R），没有T2复制时间。

可见，扩容后，Minor GC时增加了T1（扫描时间），但省去T2（复制对象）的时间，更重要的是对于虚拟机来说，复制对象的成本要远高于扫描成本，所以，单次**Minor GC时间更多取决于GC后存活对象的数量，而非Eden区的大小**。因此如果堆中短期对象很多，那么扩容新生代，单次Minor GC时间不会显著增加。下面需要确认下服务中对象的生命周期分布情况：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VkZXlNqAHPkAhPPMYzaY7dsTxsEUApedQI7O16Was6rwL6EcCPxVy2A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过上图GC日志中两处红色框标记内容可知：

1. new threshold = 2（动态年龄判断，对象的晋升年龄阈值为2），对象仅经历2次Minor GC后就晋升到老年代，这样老年代会迅速被填满，直接导致了频繁的Major GC。
2. Major GC后老年代使用空间为300M+，意味着此时绝大多数(86% = 2G/2.3G)的对象已经不再存活，也就是说生命周期长的对象占比很小。

由此可见，服务中存在大量短期临时对象，扩容新生代空间后，Minor GC频率降低，对象在新生代得到充分回收，只有生命周期长的对象才进入老年代。这样老年代增速变慢，Major GC频率自然也会降低。

#### 优化结果

通过扩容新生代为为原来的三倍，单次Minor GC时间增加小于5ms，频率下降了60%，服务响应时间TP90，TP99都下降了10ms+，服务可用性得到提升。

调整前：![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VPfprosZI3KazSC5VuD137pUZeKYnAvYwyh648ibXwTFCqdS3qGOfhAw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

调整后：![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VPFcYQjlBeg31fXUqrptib6eo1pSp9n1WCuwUpzszbv9OSby9lTez2NQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 小结

如何选择各分区大小应该依赖应用程序中**对象生命周期的分布情况：如果应用存在大量的短期对象，应该选择较大的年轻代；如果存在相对较多的持久对象，老年代应该适当增大。**

#### 更多思考

关于上文中提到晋升年龄阈值为2，很多同学有疑问，为什么设置了MaxTenuringThreshold=15，对象仍然仅经历2次Minor GC，就晋升到老年代？这里涉及到“动态年龄计算”的概念。

**动态年龄计算**：

Hotspot遍历所有对象时，按照年龄从小到大对其所占用的大小进行累积，当累积的某个年龄大小超过了survivor区的一半时，取这个年龄和MaxTenuringThreshold中更小的一个值，作为新的晋升年龄阈值。在本案例中，调优前：Survivor区 = 64M，desired survivor = 32M，此时Survivor区中age<=2的对象累计大小为41M，41M大于32M，所以晋升年龄阈值被设置为2，下次Minor GC时将年龄超过2的对象被晋升到老年代。

JVM引入动态年龄计算，主要基于如下两点考虑：

1. 如果固定按照MaxTenuringThreshold设定的阈值作为晋升条件：

   a）MaxTenuringThreshold设置的过大，原本应该晋升的对象一直停留在Survivor区，直到Survivor区溢出，一旦溢出发生，Eden+Svuvivor中对象将不再依据年龄全部提升到老年代，这样对象老化的机制就失效了。

   b）MaxTenuringThreshold设置的过小，“过早晋升”即对象不能在新生代充分被回收，大量短期对象被晋升到老年代，老年代空间迅速增长，引起频繁的Major GC。分代回收失去了意义，严重影响GC性能。

2. 相同应用在不同时间的表现不同：特殊任务的执行或者流量成分的变化，都会导致对象的生命周期分布发生波动，那么固定的阈值设定，因为无法动态适应变化，会造成和上面相同的问题。

总结来说，为了更好的适应不同程序的内存情况，虚拟机并不总是要求对象年龄必须达到Maxtenuringthreshhold再晋级老年代。

### 美团案例6: 请求高峰期发生GC，导致服务可用性下降

这个案例，来自美团技术官网

#### 确定目标

GC日志显示，高峰期CMS在重标记（Remark）阶段耗时1.39s。

Remark阶段是Stop-The-World（以下简称为STW）的，即在执行垃圾回收时，Java应用程序中除了垃圾回收器线程之外其他所有线程都被挂起，意味着在此期间，用户正常工作的线程全部被暂停下来，这是低延时服务不能接受的。本次优化目标是降低Remark时间。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VqHIM4hxiaIiaMEJWg9N0AjIHkM7DHkIB5tAaGmWHD85UOZJY1TOnIpoQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 优化

解决问题前，先回顾一下CMS的四个主要阶段，以及各个阶段的工作内容。下图展示了CMS各个阶段可以标记的对象，用不同颜色区分。

1. Init-mark初始标记(STW) ，该阶段进行可达性分析，标记GC ROOT能直接关联到的对象，所以很快。
2. Concurrent-mark并发标记，由前阶段标记过的绿色对象出发，所有可到达的对象都在本阶段中标记。
3. Remark重标记(STW) ，暂停所有用户线程，重新扫描堆中的对象，进行可达性分析，标记活着的对象。因为并发标记阶段是和用户线程并发执行的过程，所以该过程中可能有用户线程修改某些活跃对象的字段，指向了一个未标记过的对象，如下图中红色对象在并发标记开始时不可达，但是并行期间引用发生变化，变为对象可达，这个阶段需要重新标记出此类对象，防止在下一阶段被清理掉，这个过程也是需要STW的。特别需要注意一点，这个阶段是以新生代中对象为根来判断对象是否存活的。
4. 并发清理，进行并发的垃圾清理。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VPpe3FWAWyTvrkQ6DeHWp6PVcc3AQAewfvaibia6wf2fp685bu0wALgrQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可见，Remark阶段主要是通过扫描堆来判断对象是否存活。那么准确判断对象是否存活，需要扫描哪些对象？CMS对老年代做回收，Remark阶段仅扫描老年代是否可行？结论是不可行，原因如下：![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VmdL7kccBJiacMpZnlqVrKDPePbrLt8jGJ3dfsibDg9LSwicHF0Zy5l80A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果仅扫描老年代中对象，即以老年代中对象为根，判断对象是否存在引用，上图中，对象A因为引用存在新生代中，它在Remark阶段就不会被修正标记为可达，GC时会被错误回收。

新生代对象持有老年代中对象的引用，这种情况称为**“跨代引用”**。因它的存在，Remark阶段必须扫描整个堆来判断对象是否存活，包括图中灰色的不可达对象。

灰色对象已经不可达，但仍然需要扫描的原因：**新生代GC和老年代的GC是各自分开独立进行的**，只有Minor GC时才会使用根搜索算法，标记新生代对象是否可达，也就是说虽然一些对象已经不可达，但在Minor GC发生前不会被标记为不可达，CMS也无法辨认哪些对象存活，只能全堆扫描（新生代+老年代）。

由此可见堆中对象的数目影响了Remark阶段耗时。 分析GC日志可以得出同样的规律，Remark耗时>500ms时，新生代使用率都在75%以上。这样降低Remark阶段耗时问题转换成如何减少新生代对象数量。

新生代中对象的特点是“朝生夕灭”，这样如果Remark前执行一次Minor GC，大部分对象就会被回收。

CMS就采用了这样的方式，在Remark前增加了一个可中断的并发预清理（CMS-concurrent-abortable-preclean），该阶段主要工作仍然是并发标记对象是否存活，只是这个过程可被中断。

此阶段在Eden区使用超过2M时启动，当然2M是默认的阈值，可以通过参数修改。如果此阶段执行时等到了Minor GC，那么上述灰色对象将被回收，Reamark阶段需要扫描的对象就少了。

除此之外CMS为了避免这个阶段没有等到Minor GC而陷入无限等待，提供了参数CMSMaxAbortablePrecleanTime ，默认为5s，含义是如果可中断的预清理执行超过5s，不管发没发生Minor GC，都会中止此阶段，进入Remark。

根据GC日志红色标记2处显示，可中断的并发预清理执行了5.35s，超过了设置的5s被中断，期间没有等到Minor GC ，所以Remark时新生代中仍然有很多对象。

对于这种情况，CMS提供CMSScavengeBeforeRemark参数，用来保证Remark前强制进行一次Minor GC。

#### 优化结果

经过增加CMSScavengeBeforeRemark参数，单次执行时间>200ms的GC停顿消失，从监控上观察，GCtime和业务波动保持一致，不再有明显的毛刺。![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VzrEOX6dH9hE06qdwxM8G5sIoemwdxY7Zj5XJn58aF8YrZs4Sr2Lwcw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 小结

通过案例分析了解到，由于跨代引用的存在，CMS在Remark阶段必须扫描整个堆，同时为了避免扫描时新生代有很多对象，增加了可中断的预清理阶段用来等待Minor GC的发生。只是该阶段有时间限制，如果超时等不到Minor GC，Remark时新生代仍然有很多对象，我们的调优策略是，通过参数强制Remark前进行一次Minor GC，从而降低Remark阶段的时间。

#### 更多思考

案例中只涉及老年代GC，其实新生代GC存在同样的问题，即老年代可能持有新生代对象引用，所以Minor GC时也必须扫描老年代。

**JVM是如何避免Minor GC时扫描全堆的？** 经过统计信息显示，老年代持有新生代对象引用的情况不足1%，根据这一特性JVM引入了卡表（card table）来实现这一目的。如下图所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VLXicUJuexEpEkP48NpNm9BdLh6O5ZvfwQM6L7QMRWHeEtEEyWicPLdNQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**卡表**的具体策略是将老年代的空间分成大小为512B的若干张卡（card）。卡表本身是单字节数组，数组中的每个元素对应着一张卡，当发生老年代引用新生代时，虚拟机将该卡对应的卡表元素设置为适当的值。如上图所示，卡表3被标记为脏（卡表还有另外的作用，标识并发标记阶段哪些块被修改过），之后Minor GC时通过扫描卡表就可以很快的识别哪些卡中存在老年代指向新生代的引用。这样虚拟机通过空间换时间的方式，避免了全堆扫描。

总结来说，CMS的设计聚焦在获取最短的时延，为此它“不遗余力”地做了很多工作，包括尽量让应用程序和GC线程并发、增加可中断的并发预清理阶段、引入卡表等，虽然这些操作牺牲了一定吞吐量但获得了更短的回收停顿时间。

### 美团案例7：发生Stop-The-World的GC

这个案例，来自美团技术官网

#### 确定目标

GC日志如下图（在GC日志中，Full GC是用来说明这次垃圾回收的停顿类型，代表STW类型的GC，并不特指老年代GC），根据GC日志可知本次Full GC耗时1.23s。这个在线服务同样要求低时延高可用。

本次优化目标是降低单次STW回收停顿时间，提高可用性。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VzHlTyxqL0Wu6wj4mqoBjcC2YdpGgkwWtyr2emiaSVGgEWu1y5rFX4YA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 优化

首先，什么时候可能会触发STW的Full GC呢？

1. Perm空间不足；
2. CMS GC时出现promotion failed和concurrent mode failure（concurrent mode failure发生的原因一般是CMS正在进行，但是由于老年代空间不足，需要尽快回收老年代里面的不再被使用的对象，这时停止所有的线程，同时终止CMS，直接进行Serial Old GC）；
3. 统计得到的Young GC晋升到老年代的平均大小大于老年代的剩余空间；
4. 主动触发Full GC（执行jmap -histo:live [pid]）来避免碎片问题。

然后，我们来逐一分析一下：

- 排除原因2：如果是原因2中两种情况，日志中会有特殊标识，目前没有。
- 排除原因3：根据GC日志，当时老年代使用量仅为20%，也不存在大于2G的大对象产生。
- 排除原因4：因为当时没有相关命令执行。
- 锁定原因1：根据日志发现Full GC后，Perm区变大了，推断是由于永久代空间不足容量扩展导致的。

找到原因后解决方法有两种：

1. 通过把-XX:PermSize参数和-XX:MaxPermSize设置成一样，强制虚拟机在启动的时候就把永久代的容量固定下来，避免运行时自动扩容。
2. CMS默认情况下不会回收Perm区，通过参数CMSPermGenSweepingEnabled、CMSClassUnloadingEnabled ，可以让CMS在Perm区容量不足时对其回收。

由于该服务没有生成大量动态类，回收Perm区收益不大，所以我们采用方案1，启动时将Perm区大小固定，避免进行动态扩容。

#### 优化结果

调整参数后，服务不再有Perm区扩容导致的STW GC发生。

#### 小结

对于性能要求很高的服务，建议将MaxPermSize和MinPermSize设置成一致（JDK8开始，Perm区完全消失，转而使用元空间。而元空间是直接存在内存中，不在JVM中），Xms和Xmx也设置为相同，这样可以减少内存自动扩容和收缩带来的性能损失。虚拟机启动的时候就会把参数中所设定的内存全部化为私有，即使扩容前有一部分内存不会被用户代码用到，这部分内存在虚拟机中被标识为虚拟内存，也不会交给其他进程使用。

## 八：JVM调优常见面试题的精简答案

#### 8. 1、调优包括哪些维度？

架构调优、代码调优、JVM调优、数据库调优、操作系统调优等

架构调优和代码调优是JVM调优的基础，其中**架构调优是对系统影响最大的**

#### 8.2、何时进行JVM调优

- Heap内存（老年代）持续上涨达到设置的最大内存值；
- Full GC 次数频繁；
- GC 停顿时间过长（超过1秒）；
- 应用出现OutOfMemory等内存异常；
- 应用中有使用本地缓存且占用大量内存空间；
- 系统吞吐量与响应性能不高或不降；

#### 8.3、JVM调优的基本原则

- 大多数的Java应用不需要进行JVM优化；
- 大多数导致GC问题的原因是代码层面的问题导致的（代码层面）；
- 上线之前，应先考虑将机器的JVM参数设置到最优；
- 减少创建对象的数量（代码层面）；
- 减少使用全局变量和大对象（代码层面）；
- 优先架构调优和代码调优，JVM优化是不得已的手段（代码、架构层面）；
- 分析GC情况优化代码比优化JVM参数更好（代码层面）

**其实最有效的优化手段是架构和代码层面的优化，而JVM优化则是最后不得已的手段，也可以说是对服务器配置的最后一次“压榨”**

#### 8.4、JVM调优目标

目的都是为了令应用程序使用最小的硬件消耗来承载更大的吞吐。JVM调优主要是针对垃圾收集器的收集性能优化，令运行在虚拟机上的应用能够使用更少的内存以及延迟获取更大的吞吐量，总结以下：

- 延迟：GC低停顿和GC低频率；
- 低内存占用；
- 高吞吐量。

#### 8.5、JVM调优量化目标

- Heap 内存使用率 <= 70%;
- Old generation 内存使用率 <= 70%;
- avgpause <= 1秒;
- Full GC 次数 0 或 avg pause interval >= 24小时。

#### 8.6、JVM调优的步骤

- 分析GC日志及dump文件，判断是否需要优化，确定瓶颈问题点；
- 确定JVM调优量化目标；
- 确定JVM调优参数（根据历史JVM参数来调整）；
- 依次调优内存、延迟、吞吐量等指标；
- 对比观察调优前后的差异；
- 不断的分析和调整，直到找到合适的JVM参数配置；
- 找到最合适的参数，将这些参数应用到所有服务器，并进行后续跟踪。

#### 8.7、VM参数解析及调优

```
-Xmx4g 
–Xms4g 
–Xmn1200m 
–Xss512k 
-XX:NewRatio=4 
-XX:SurvivorRatio=8 
-XX:PermSize=100m 
-XX:MaxPermSize=256m 
-XX:MaxTenuringThreshold=15
```

- -Xmx4g：堆内存最大值为4GB。
- -Xms4g：初始化堆内存大小为4GB。
- -Xmn1200m：**设置年轻代大小为1200MB**。增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。
- -Xss512k：**设置每个线程的堆栈大小**。JDK5.0以后每个线程堆栈大小为1MB，以前每个线程堆栈大小为256K。应根据应用线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。
- -XX:NewRatio=4：**设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）**。设置为4，则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5
- -XX:SurvivorRatio=8：**设置年轻代中Eden区与Survivor区的大小比值**。设置为8，则两个Survivor区与一个Eden区的比值为2:8，一个Survivor区占整个年轻代的1/10
- -XX:PermSize=100m：初始化永久代大小为100MB。
- -XX:MaxPermSize=256m：设置持久代大小为256MB。
- -XX:MaxTenuringThreshold=15：**设置垃圾最大年龄**。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论。

**可调优参数：**

- -Xms：初始化堆内存大小，默认为物理内存的1/64(小于1GB)。
- -Xmx：**堆内存最大值**。默认(MaxHeapFreeRatio参数可以调整)空余堆内存大于70%时，JVM会减少堆直到-Xms的最小限制。
- -Xmn：新生代大小，包括Eden区与2个Survivor区。
- -XX:SurvivorRatio=1：Eden区与一个Survivor区比值为1:1。
- -XX:MaxDirectMemorySize=1G：**直接内存**。报java.lang.OutOfMemoryError: Direct buffer memory异常可以上调这个值。
- -XX:+DisableExplicitGC：禁止运行期显式地调用System.gc()来触发fulll GC。
- 注意: Java RMI的定时GC触发机制可通过配置-Dsun.rmi.dgc.server.gcInterval=86400来控制触发的时间。
- -XX:CMSInitiatingOccupancyFraction=60：老年代内存回收阈值，默认值为68。
- -XX:ConcGCThreads=4：CMS垃圾回收器并行线程线，推荐值为CPU核心数。
- -XX:ParallelGCThreads=8：新生代并行收集器的线程数。
- -XX:MaxTenuringThreshold=10：**设置垃圾最大年龄**。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论。
- -XX:CMSFullGCsBeforeCompaction=4：指定进行多少次fullGC之后，进行tenured区 内存空间压缩。
- -XX:CMSMaxAbortablePrecleanTime=500：当abortable-preclean预清理阶段执行达到这个时间时就会结束。

#### 8.8、内存调优示例

```
-XX:+PrintGC 　　输出GC日志
-XX:+PrintGCDetails 输出GC的详细日志
-XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）
-XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
-XX:+PrintHeapAtGC 　　在进行GC的前后打印出堆的信息
-Xloggc:../logs/gc.log 日志文件的输出路径
```

#### ![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WOUj0gsHiaMETz9cKMUHYr6VO0eFaaKXoubGPOicdoC3Vxg8iakMibrTiaicHqulNPvFGOxXuqyE6Cia9dQw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- java heap：参数-Xms和-Xmx，建议扩大至3-4倍FullGC后的老年代空间占用。

- 永久代：-XX:PermSize和-XX:MaxPermSize，建议扩大至1.2-1.5倍FullGc后的永久代空间占用。

- 新生代：-Xmn，建议扩大至1-1.5倍FullGC之后的老年代空间占用。

- 老年代：2-3倍FullGC后的老年代空间占用。

  ```
  -Xms373m -Xmx373m //4*93=372
  -Xmn140m //1.5*93=139.5
  -XX:PermSize=5m -XX:MaxPermSize=5m //1.5*3=4.5
  ```

## 九 、结语

在JVM调优中，关键在于准确识别系统的性能瓶颈和优化方向，选择适合的调优策略和参数。

实施调优方案后，必须验证效果，并持续监控系统性能，及时调整优化策略和参数以保持系统高性能和稳定性。

同时，需要及时发现和解决各种潜在的性能问题，如内存泄漏、CPU飙升、频繁的垃圾回收等，以确保系统在高负载和复杂环境下能够保持卓越的性能表现。

总之，JVM调优是一个持续改进的过程，通过对系统性能的深入分析和优化，确保Java应用程序在各种情况下都能够保持高效稳定的运行状态。

随着硬件技术的迅速发展，JVM调优也将面临新的挑战和机遇。新一代的处理器、存储技术以及分布式系统架构等将对JVM调优提出更高的要求，需要更智能、更高效的优化方案来适应日益复杂的应用场景和巨大的数据处理需求。

未来，JVM调优将持续创新和进步，以满足不断变化的业务需求和技术挑战，为Java应用程序提供更稳定、更高效的运行环境，推动Java生态系统的蓬勃发展和壮大。

与开篇所述保持一致，我们强调在JVM调优中，真正的参数调整是较少的，更多的是通过分析日志和结合系统业务进行代码层面的优化。

这可能是调优工作中占据更大比重的内容。我们不应迷失方向，只为了调优而调优，只为了调整参数而调整参数。最终，我们需要回归到业务本质，这才是最核心的内容。我们也需要更深入地了解JVM的相关参数，以更好地支撑业务需求的实现。
