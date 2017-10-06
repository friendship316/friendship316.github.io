---
title: 《深入理解Java虚拟机》读书笔记(1)——虚拟机结构与垃圾回收
date: 2017-09-04
---

简单记录一些《深入理解Java虚拟机》的笔记（基本所有图和话都摘自《深入理解Java虚拟机》）

###  第二章
#### 1、虚拟机运行时结构

![虚拟机运行时结构](http://upload-images.jianshu.io/upload_images/3727888-4ba5fd8db8146b96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**程序计数器：**可看做当前线程所执行的字节码的行号指示器。

**虚拟机栈：**描述java方法执行的内存模型。每个方法在执行的同时都会创建一个栈帧（ Stack Frame） 用于存储局部变量表、 操作数栈、 动态链接、 方法出口等信息。 其中局部变量表（主要关注点）存放了编译期可知的基本数据类型、对象引用和returnAddress类型（ 指向了 一条字节码指令的地址）。

**本地方法栈：**与虚拟机栈作用类似，它们之间的区别不过是虚拟机栈为虚拟机执行Java方法（ 也就是字节码） 服务， 而本地方法栈则为虚拟机使用到的Native方法服务。有的虚拟机（ 如Sun HotSpot虚拟机） 直接就把本地方法栈和虚拟机栈合二为一。 

**Java堆：**虚拟机启动时创建，存储对象实例，垃圾收集器管理的主要区域。Java堆中还可以细分为： 新生代和老年代； 新生代中再细致一点的有Eden空间、 From Survivor空间、 To Survivor空间。

**方法区：**用于存储已被虚拟机加载的类信息、 常量、 静态变量、 即时编译器编译后的代码等数据。 运行时常量池（ Runtime Constant Pool） 是方法区的一部分。String.intern() 是一个Native方法， 它的作用是： 如果字符串常量池中已经包含一个等于此String对象的字符串， 则返回代表池中这个字符串的String对象； 否则， 将此String对象包含的字符串添加到常量池中， 并且返回此String对象的引用。

**本机直接内存：**不是虚拟机运行时数据区的一部分， 也不是Java虚拟机规范中定义的内存区域。NIO（ New Input/Output） 类， 引入了 一种基于通道（ Channel） 与缓冲区（ Buffer） 的I/O方式， 它可以使用 Native函数库直接分配堆外内存， 然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。 这样能在一些场景中显著提高性能， 因为避免了 在Java堆和Native堆中来回复制数据。默认与堆最大值“-Xmx”相同。

### 第三章
#### 1、三种垃圾回收算法

##### A、标记-清除算法

标记-清除算法分为 “标记”和“清除”两个阶段： 首先标记出所有需要回收的对象， 在标记完成后统一回收所有被标记的对象。它的主要不足有两个： 一个是效率问题， 标记和清除两个过程的效率都不高； 另一个是空间问题， 标记清除之后会产生大量不连续的内存碎片， 空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时， 无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。 

![标记-清除算法示意图](http://upload-images.jianshu.io/upload_images/3727888-a1fba6f4597bc9c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### B、复制算法

将可用内存按容量划分为大小相等的两块， 每次只使用其中的一块。 当这一块的内存用完了 ， 就将还存活着的对象复制到另外一块上面， 然后再把已使用过的内存空间一次清理掉。 这样使得每次都是对整个半区进行内存回收， 内存分配时也就不用考虑内存碎片等复杂情况， 只要移动堆顶指针， 按顺序分配内存即可， 实现简单， 运行高效。 只是这种算法的代价是将内存缩小为了 原来的一半， 未免太高了 一点。

但现在的商业虚拟机都采用这种收集算法来回收新生代， IBM公司的专门研究表明， 新生代中的对象98%是“朝生夕死”的， 所以并不需要按照1:1的比例来划分内存空间， 而是将内存分为一块较大的Eden空间和两块较小的Survivor空间， 每次使用 Eden和其中一块Survivor。当回收时， 将Eden和Survivor中还存活着的对象一次性地复制到另外一块Survivor空间上， 最后清理掉Eden和刚才用过的Survivor空间。 HotSpot虚拟机默认Eden和Survivor的大小比例是8:1， 也就是每次新生代中可用内存空间为整个新生代容量的90%（ 80%+10%） ， 只有10%的内存会被“浪费”。 当然， 98%的对象可回收只是一般场景下的数据， 我们没有办法保证每次回收都只有不多于10%的对象存活， 如果另外一块Survivor空间没有足够空间存放上一次新生代收集下来的存活对象时，这些对象将直接通过分配担保机制进入老年代。

![复制算法示意图](http://upload-images.jianshu.io/upload_images/3727888-1b21aca79a74d4c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### C、标记整理算法
标记整理算法的标记过程仍然与“标记-清除”算法一样， 但后续步骤不是直接对可回收对象进行清理， 而是让所有存活的对象都向一端移动， 然后直接清理掉端边界以外的内存。

![标记整理算法示意图](http://upload-images.jianshu.io/upload_images/3727888-8aba428d138444d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**结论**
以上三种算法各有优缺点，所以在实际应用中，形成了现在的“分代收集算法”，根据对象存活周期的不同将内存划分为几块。 一般是把Java堆分为新生代和老年代， 这样就可以根据各个年代的特点采用最适当的收集算法。 在新生代中， 每次垃圾收集时都发现有大批对象死去， 只有少量存活， 那就选用复制算法， 只需要付出少量存活对象的复制成本就可以完成收集。 而老年代中因为对象存活率高、 没有额外空间对它进行分配担保， 就必须使用 “标记—清理”或者“标记—整理”算法来进行回收。

#### 2、垃圾收集器
JDK 1.7 Update 14（及以后的版本）包含了7个收集器。对于垃圾收集器的总结，粗略如下图。图中STW=StopThe World，即暂停所有工作线程。并行是指有多个线程进行垃圾收集。而并发是指垃圾收集的线程和工作线程同时运行。两个收集器中的连线是代表这两个收集器可以组合使用（开启参数见下方附录）。

![多种垃圾收集器示意图](http://ouws94yej.bkt.clouddn.com/xunijilajishoujiqi.png)

CMS（Concurrent Mark Sweep）收集器和G1（Garbage-First）收集器拥有了并发能力，所以整个收集过程比其余收集器复杂一些，标记过程分为多个阶段，特别记录一下。

CMS收集器运行示意图：

![CMS收集器运行示意图](http://upload-images.jianshu.io/upload_images/3727888-61e398f4eceb2130.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

G1收集器运行示意图：

![G1收集器运行示意图](http://upload-images.jianshu.io/upload_images/3727888-9f1a3b62e10742cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 附录

记录下从书中了解的一些常见虚拟机配置参数，便于查阅：

#### 1\. 配置调整类型参数 

|参数名   |意义    | 示例 | 备注|
| :---: | :--------: | :-----: | :---: |
| -Xms | 堆的最小值 | -Xms512M | 单位：M（m） |
| -Xmx | 堆的最大值 | -Xmx1024M | 单位：M（m） |
| -Xmn | 年轻代大小 | -Xmn100M | 单位：M（m） |
| -Xss | 栈内存大小 | -Xss100M | 单位：M（m） |
| -Xoss  |  本地方法栈大小  | -Xoss100M  |  对于HotSpot无效  |
| -XX:PermSize  | 方法区大小 | -XX:PermSize=128M  | 单位：M（m）  |
| -XX:MaxPermSize  | 方法区最大容量 | -XX:MaxPermSize=256M  | 单位：M（m）  |
| -XX:MaxDirectMemorySize  |  本机直接内存容量 | -XX:MaxDirectMemorySize =256M  | 单位：M（m）  |
| -XX:SurvivorRatio  | 新生代中Eden和survivor的比例  |  -XX:SurvivorRatio=8 | 默认为8  |
| -XX:ParallelGCThreads| 并行GC时进行内存回收的线程数| -XX:ParallelGCThreads=43 | |
| -XX:PretenureSizeThreshold | 直接晋升老年代的对象大小|-XX:PretenureSizeThreshold=3145728|单位：字节
| -XX:MaxTenuringThreshold| 晋升到老年代的对象年龄|-XX:MaxTenuringThreshold=1|每经过一次GC，年龄+1
| -XX:GCTimeRatio | GC时间占总时间的比率 | -XX:GCTimeRatio=99 | 默认值为99，即允许1%的GC时间，仅对Parallel Scavenge 生效|
| -XX:MaxGCPauseMillis | 设置GC的最大停顿时间| -XX:MaxGCPauseMillis |  仅对Parallel Scavenge 生效,单位：毫秒 |
| -XX:CMSInitiatingOccupancyFraction| 设置老年代空间被使用多少后触发垃圾回收（百分比）| -XX:CMSInitiatingOccupancyFraction=85 | 仅对CMS收集器有效|
|-XX:CMSFullGCsBeforeCompaction|设置在进行若干次垃圾收集后再启动一次内存碎片整理|-XX:CMSFullGCsBeforeCompaction=0 | 仅对CMS有效,默认值为0|

#### 2\. 开关类型的参数：

|参数名   |意义    |  备注 |
| :---: | :--------: |  :-------------: |
| -XX:+HeapDumpOnOutOfMemoryError  | 让虚拟机在出现内存溢出时Dump出当前的内存堆转储快照  |  快照可用诸如Eclipse Memory Analyzer这类工具对快照进行分析 |
|  -Xnoclassgc |  控制是否对类进行回收 |   |  
| -XX: +PrintGCDetails  | 收集GC日志  |   | 
|  -XX:+TraceClassLoading |  查看类加载信息 | Product版的虚拟机*  | 
| -XX:+TraceClassLoading  |  查看类卸载信息  | FastDebug版的虚拟机*  | 
|  -XX:+UseSerialGC  | 使用Serial+Serial Old的收集器组合进行内存回收  |   |
| -XX:+UserParNewGC  |  使用ParNew+Serial Old的收集器组合进行内存回收 |   |
| -XX:+UseConcMarkSweepGC  |  使用ParNew+CMS+Serial Old的收集器组合进行内存回收 | 此时Serial Old作为CMS的后备收集器   |
| -XX:+UseParallelGC  | 使用Parallel Scavenge+Serial Old的收集器组合进行内存回收   |   |
| -XX:+UseParallelOldGC  |  使用Parallel Scavenge+Parallel Old的收集器组合进行内存回收  |   |
| -XX:+UseAdaptiveSizePolicy  |  动态调整Java堆中各个区域的大小以及进入老年代的年龄 |   |
| -XX:+HandlePromotionFailure| 是否允许分配担保失败  |   |
|  -XX:+UseCMSCompactAtFullCollection| 设置完成垃圾收集后是否要进行一次内存碎片整理| 仅对CMS有效|
