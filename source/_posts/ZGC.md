---
title: ZGC
tags:
  - Java
  - JVM垃圾回收器
typora-root-url: ../../themes/butterfly/source
date: 2020-11-25 11:23:30
description: 之前我写了各种垃圾回收器，但是对于Java11里的两个新垃圾回收器介绍的比较简单，虽然现在大部分公司用的还是Java8，但是提前了解一下，学习一下先进技术，也可以知识迁移到其他地方去解决其他问题。
cover: /blogImg/ZGC工作流程.png
categories: Java基础
---

现代垃圾收集器的演进大部分都是往减少停顿方向发展。

像 CMS 就是分离出一些阶段使得应用线程可以和垃圾回收线程并发，当然还有利用回收线程的并行来减少停顿的时间。

基本上 STW 阶段都是利用多线程并行来减少停顿时间，而并发阶段不会有太多的回收线程工作，这是为了不和应用线程争抢 CPU，反正都并发了慢就慢点（不过还是得考虑内存分配速率）。

而 G1 可以认为是打开了另一个方向的大门：只回收部分垃圾来减少停顿时间。

不过为了达到只回收部分 reigon，每个 region 都需要 RememberSet 来记录各 region 之间的引用。这个内存的开销其实还是挺大的，可能会占据整堆的20%或以上。

并且 G1 还有写屏障的开销，虽说用了 logging wtire barrier，但也还是有开销的。

当然 CMS 也用了写屏障，不过逻辑比较简单，啥都没判断就单纯的记录。

其实 G1 相对于 CMS 只有在大堆的场景下才有优势，CMS 比较伤的是 remark 阶段，如果堆太大需要扫描的东西太多。

而 G1 在大堆的时候可以选择部分收集，所以停顿时间有优势。

今天的主角 ZGC 和 G1 一样是基于 reigon 的，几乎所有阶段都是并发的，整堆扫描，部分收集。

而且 ZGC 还不分代，就是没分新生代和老年代。

# ZGC的目标

垃圾收集器设计出来都有目标的，有些是为了更高的吞吐，有些是为了更低的延迟。

所以我们先看看 ZGC 的目标：

![ZGC的目标](/blogImg/ZGC的目标.png)

可以看到它的目标就是低延迟，保证最大停顿时间在几毫秒之内，不管你堆多大或者存活的对象有多少。

可以处理 8MB-16TB 的堆。

咱们就按 openjdk 的 wiki 来展开今天的内容。

![ZGC关键字](/blogImg/ZGC关键字.png)

关键字：并发、基于Region、整理内存、支持NUMA、用了染色指针、用了读屏障，对了 ZGC 用的是 STAB。

## Concurrent

这个 Concurrent 的意思是和应用线程并发执行，ZGC 一共分了 10 个阶段，只有 3 个很短暂的阶段是 STW 的。

![ZGC工作流程](/blogImg/ZGC工作流程.png)

可以看到只有初始标记、再标记、初始转移阶段是 STW 的。

初始标记就扫描 GC Roots 直接可达的，耗时很短，重新标记一般而言也很短，如果超过 1ms 会再次进入并发标记阶段再来一遍，所以影响不大。

初始转移阶段也是扫描 GC Roots 也很短，所以可以认为 ZGC 几乎是并发的。

而且之所以说停顿时间不会随着堆的大小和存活对象的数量增加而增加，是因为 STW 几乎只和 GC Roots 集合大小有关，和堆大小没啥关系。

这其实就是 ZGC 超过 G1 很关键的一个地方， G1 的对象转移需要 STW 所以堆大需要转移对象多，停顿的时间就长了，而 ZGC 有并发转移。

不过并发回收有个情况就是回收的时候应用线程还是在产生新的对象，所以需要预留一些空间给并发时候生成的新对象。

如果对象分配过快导致内存不够，在 CMS 中是发生 Full gc，而 ZGC 则是阻塞应用线程。

所以要注意 ZGC 触发的时间。

ZGC 有自适应算法来触发也有固定时间触发，所以可以根据实际场景来修改 ZGC 触发时间，防止过晚触发而内存分配过快导致线程阻塞。

还有设置 ParallelGCThreads 和 ConcGCThreads，分别是 STW 并行时候的线程数和并发阶段的线程数来加快回收的速度。

不过 ConcGCThreads 数量需要注意，因为此阶段是和应用线程并发，如果线程数过多会影响应用线程。

其实 ZGC 的每个阶段都是串行的，所以理论上其实可以不需要分两类线程，那为什么分了这两类线程？

就是为了灵活设置。分成两类就可以通过配置来调优，达到性能最大值。

对了上面提到 ZGC 的 STW 和 GC Roots 集合大小有关系，所以如果在会生成很多线程、动态加载很多 ClassLoader 等情况下会增加 ZGC 的停顿时间。

## Region-based

为了能更细粒度的控制内存的分配，和 G1 一样 ZGC 也将堆划分成很多分区。

分了三种：2MB、32MB 和 X*MB（受操作系统控制）。

下图为源码中的注释：

![Region-based](/blogImg/Region-based.png)

对于回收的策略是优先收集小区，中、大区尽量不回收。

## Compacting

和 G1 一样都分区了所以肯定从整体来看像是标记-复制算法，所以也是会整理的。

因此 ZGC 也不会产生内存碎片。

具体的流程下文再做分析。

## NUMA-aware

以前的 G1 是不支持的，不过在 JDK14 G1 也支持了。

![NUMA-aware](/blogImg/NUMA-aware.png)

可能有的同学对 NUMA 不太熟悉，没事我先来解释一波。

在早期处理器都是单核的，因为根据摩尔定律，处理器的性能每隔一段时间就可以成指数型增长。

而近年来这个增长的速度逐渐变缓，于是很多厂商就推出了双核多核的计算机。

早期 CPU 通过前端总线到北桥到内存总线然后才访问到内存。

![SMP](/blogImg/SMP.png)

这个架构被称为 SMP （Symmetric Multi-Processor），因为任一个 CPU 对内存的访问速度是一致的，不用考虑不同内存地址之间的差异，所以也称一致内存访问（Uniform Memory Access， UMA ）。

这个核心越加越多，渐渐的总线和北桥就成为瓶颈，那不能够啊，于是就想了个办法。

把 CPU 和内存集成到一个单元上，这个就是非一致内存访问 （Non-Uniform Memory Access，NUMA）。

![NUMA](/blogImg/NUMA.png)

简单的说就是把内存分一分，每个 CPU 访问自己的本地的内存比较快，访问别人的远程内存就比较慢。

当然也可以多个 CPU 享受一块内存或者多块，如下图所示：

![多个 CPU 共享一块内存或者多块](/blogImg/多个 CPU 共享一块内存或者多块.png)

但是因为内存被切分为本地内存和远程内存，当某个模块比较“热”的时候，就可能产生本地内存爆满，而远程内存都很空闲的情况。

比如 64G 内存一分为二，模块一的内存用了31G，而另一个模块的内存用了5G，且模块一只能用本地内存，这就产生了内存不平衡问题。

如果有些策略规定不能访问远程内存的时候，就会出现明明还有很多内存却产生 SWAP(将部分内存置换到硬盘中) 的情况。

即使允许访问远程内存那也比本地内存访问速率相差较大，这是使用 NUMA 需要考虑的问题。

ZGC 对 NUMA 的支持是小分区分配时会优先从本地内存分配，如果本地内存不足则从远程内存分配。

对于中、大分区的话就交由操作系统决定。

上述做法的原因是生成的绝大部分都是小分区对象，因此优先本地分配速度较快，而且也不易造成内存不平衡的情况。

而中、大分区对象较大，如果都从本地分配则可能会导致内存不平衡的情况。

## Using colored pointers

染色指针其实就是从 64 位的指针中，拿几位来标识对象此时的情况，分别表示 Marked0、Marked1、Remapped、Finalizable。

![染色指针](/blogImg/染色指针.png)

我们再来看下源码中的注释，非常的清晰直观：

![染色指针注释](/blogImg/染色指针注释.png)

0-41 这 42 位就是正常的地址，所以说 ZGC 最大支持 4TB (理论上可以16TB)的内存，因为就 42 位用来表示地址。

也因此 ZGC 不支持 32 位指针，也不支持指针压缩。

然后用 42-45 位来作为标志位，其实不管这个标志位是啥指向的都是同一个对象。

这是通过多重映射来做的，很简单就是多个虚拟地址指向同一个物理地址，不过对象地址是 0001.... 还是0010....还是0100..... 对应的都是同一个物理地址即可。

具体这几个标记位怎么用的，待下文回收流程分析再解释。

不过这里先提个问题，为什么就支持 4TB，不是还有很多位没用吗？

首先 X86_64 的地址总线只有 48 条 ，所以最多其实只能用 48 位，指令集是 64 位没错，但是硬件层面就支持 48 位。

因为基本上没有多少系统支持这么大的内存，那支持 64 位就没必要了，所以就支持到 48 位。

那现在对象地址就用了 42 位，染色指针用了 4 位，不是还有 2 位可以用吗？

是的，理论上可以支持 16 TB，不过暂时认为 4TB 够了，所以暂做保留，仅此而已没啥特别的含义。

## Using load barriers

在 CMS 和 G1 中都用到了写屏障，而 ZGC 用到了读屏障。

写屏障是在对象引用赋值时候的 AOP，而读屏障是在读取引用时的 AOP。

比如 `Object a = obj.foo;`，这个过程就会触发读屏障。

也正是用了读屏障，ZGC 可以并发转移对象，而 G1 用的是写屏障，所以转移对象时候只能 STW。

简单的说就是 GC 线程转移对象之后，应用线程读取对象时，可以利用读屏障通过指针上的标志来判断对象是否被转移。

如果是的话修正对象的引用，按照上面的例子，不仅 a 能得到最新的引用地址，obj.foo 也会被更新，这样下次访问的时候一切都是正常的，就没有消耗了。

下图展示了读屏障的效果，其实就是转移的时候找地方记一下即 forwardingTable，然后读的时候触发引用的修正。

![ZGC中的读屏障](/blogImg/ZGC中的读屏障.png)

这种也称之为“自愈”，不仅赋值的引用时最新的，自身引用也修正了。

染色指针和读屏障是 ZGC 能实现并发转移的关键所在。

# ZGC 回收流程解析

ZGC 的步骤大致可分为三大阶段分别是标记、转移、重定位。

- 标记：从根开始标记所有存活对象
- 转移：选择部分活跃对象转移到新的内存空间上
- 重定位：因为对象地址变了，所以之前指向老对象的指针都要换到新对象地址上。

并且这三个阶段都是并发的。

这是意识上的阶段，具体的实现上重定位其实是糅合在标记阶段的。

在标记的时候如果发现引用的还是老的地址则会修正成新的地址，然后再进行标记。

简单的说就是从第一个 GC 开始经历了标记，然后转移了对象，这个时候不会重定位，只会记录对象都转移到哪里了。

在第二个 GC 开始标记的时候发现这个对象是被转移了，然后发现引用还是老的，则进行重定位，即修改成新的引用。

所以说重定位是糅合在下一步的标记阶段中。

![ZGC重定位](/blogImg/ZGC重定位.png)

我再简单说一下十个步骤。

不过步骤里有些不影响整体回收流程的，我就不多加分析了。

这篇文章的目的不是深入 ZGC 实现的细节，而是了解 ZGC 大致的突出点和简单流程即可。

因此想知道细节的自行查阅，或者可以看看我文末推荐的书籍。

这几个步骤主要参考文末提到的 ZGC 那本书，源码没看过所以不知道是否有出入，不过影响大，大致应该不会变。

### 初始标记

这个阶段其实大家应该很熟悉，CMS、G1 都有这个阶段，这个阶段是 STW 的，仅标记根直接可达的对象，压到标记栈中。

当然还有其他动作，比如重置 TLAB、判断是否要清除软引用等等，不做具体分析。

### 并发标记

就是根据初始标记的对象开始并发遍历对象图，还会统计每个 region 的存活对象的数量。

这个并发标记其实有个细节，标记栈其实只有一个，但是并发标记的线程有多个。

为了减少之间的竞争每个线程其实会分到不同的标记带来执行。

你就理解为标记栈被分割为好几块，每个线程负责其中的一块进行遍历标记对象，就和1.7 Hashmap 的segment 一样。

那肯定有的线程标记的快，有的标记的慢，那么先空闲下来的线程会去窃取别人的任务来执行，从而实现负载均衡。

看到这有没有想到啥？没错就是 ForkJoinPool 的工作窃取机制！

### 再标记阶段

这一阶段是 STW 的，因为并发阶段应用线程还是在运行的，所以会修改对象的引用导致漏标的情况。

因此需要个再标记阶段来标记漏标的那些对象。

如果这个阶段执行的时间过长，就会再次进入到并发标记阶段，因为 ZGC 的目标就是低延迟，所以一有高延迟的苗头就得扼制。

这个阶段还会做非强根并行标记，非强根指的是：系统字典、JVMTI、JFR、字符串表。

有些非强根可以并发，有些不行，具体不做分析。

### 非强引用并发标记和引用并发处理

就是上一步非强根的遍历，然后引用就软引用、弱引用、虚引用的一些处理。

这个阶段是并发的。

### 重置转移集

还记得标记时候的重定位么？在写读屏障时候提到的 forwardingTable 就是个映射集，你可以理解为 key 就是对象转移前的地址，value 是对象转移后的地址。

不过这个映射集在标记阶段已经用了，也就是说标记的时候已经重定位完了，所以现在没用了。

但新一轮的垃圾回收需要还是要用到这个映射集的。

因此在这个阶段对那些转移分区的地址映射集做一个复位的操作。

### 回收无效分区

回收那些物理内存已经被释放的无效的虚拟内存页面。

就是在内存紧张的时候会释放物理内存，如果同时释放虚拟空间的话也不能释放分区，因为分区需要在新一轮标记完成之后才能释放。

所以就会有无效的虚拟内存页面存在，在这个阶段回收。

### 选择待回收的分区

这和 G1 一样，因为会有很多可以回收的分区，会筛选垃圾较多的分区，来作为这次回收的分区集合。

### 初始化待转移集合的转移表

这一步就是初始化待回收的分区的 forwardingTable。

### 初始转移

这个阶段其实就是从根集合出发，如果对象在转移的分区集合中，则在新的分区分配对象空间。

如果不在转移分区集合中，则将对象标记为 Remapped。

注意这个阶段是 STW，只转移根直接可达的对象。

### 并发转移

这个阶段和并发标记阶段就很类似了，从上一步转移的对象开始遍历，做并发转移。

这一步很关键。

G1 的转移对象整体都需要 STW，而 ZGC 做到了并发转移，所以延迟会低很多。

至此十个步骤就完毕了，一次 GC 结束。

可以还能同学对染色指针的几个标记位有点懵，没事看了下文就懂了。

## 染色指针的标记位

来分析下几个标记位，M0、M1、Remapped。

先来介绍个名词，地址视图：指的就是此时地址指针的标记位。

比如标记位现在是 M0，那么此时的视图就是 M0 视图。

在垃圾回收开始前视图是 Remapped 。

在进入标记标记时。

标记线程访问发现对象地址视图是 Remapped 这时候将指针标记为 M0，即将地址视图置为 M0，表示活跃对象。

如果扫描到对象地址视图是 M0 则说明这个对象是标记开始之后新分配的或者已经标记过的对象，所以无需处理。

应用线程 如果创建新对象，则将其地址视图置为 M0，如果访问的对象地址视图是 Remapped 则将其置为 M0，并且递归标记其引用的对象。

如果访问到的是 M0 ，则无需操作。

标记阶段结束后，ZGC 会使用一个对象活跃表来存储这些对象地址，此时活跃的对象地址视图是 M0。

并发转移阶段，地址视图被置为 Remapped 。

也就是说 GC 线程如果访问到对象，此时对象地址视图是 M0，并且存在或活跃表中，则将其转移，并将地址视图置为 Remapped 。

如果在活跃表中，但是地址视图已经是 Remapped 说明已经被转移了，不做处理。

应用线程此时创建新对象，地址视图会设为  Remapped 。

此时访问对象如果对象在活跃表中，且地址视图为 Remapped  说明转移过了，不做处理。

如果地址视图为 M0，则说明还未转移，则需要转移，并将其地址视图置为 Remapped  。

如果访问到的对象不在活跃表中，则不做处理。

那 M1 什么用？

M1 是在下一次 GC 时候用的，下一次的 GC 就用 M1来标记，不用 M0。

再下一次再换过来。

简单的说就是 M1 标识本次垃圾回收中活跃的对象，而 M0 是上一次回收被标记的对象，但是没有被转移，在本次回收中也没有被标记活跃的对象。

其实从上面的分析以及得知，如果没有被转移就会停留在 M0 这个地址视图。

而下一次 GC 如果还是用 M0 来标识那混淆了这两种对象。

所以搞了个 M1。

至此染色指针这几个标志位应该就很清晰了，我在用图来示意一下。

![ZGC标记位变更](/blogImg/ZGC标记位变更.png)

