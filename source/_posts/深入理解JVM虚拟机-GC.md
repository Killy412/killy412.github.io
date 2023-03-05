---
title: 深入理解JVM虚拟机-GC
date: 2021-01-19 21:07:53
tags:
  - "JVM"
  - "GC"
categories: "JVM"
---

## 概述

> 垃圾收集需要完成的三件事
>
> - 哪些内存需要回收?
> - 什么时候回收?
> - 如何回收?

<!--more-->

## 判断对象是否存活

在进行垃圾回收之前,首先要确定哪些对象是可以回收的,哪些是存活的. 判定方法主要是两种

- 引用计数回收算法: 为对象添加一个计数器,被引用则+1,引用失效则-1. 计数器为 0 时就不能再被使用.但是如果两个对象相互引用,则计数器永远不为 0. 无法解决相互引用的问题.

- 可达性分析算法: 通过一个根节点 Gc roots 作为起始节点集,然后根据引用关系向下搜索,搜索过程所走的路径称为引用链,如果某个对象到 gc roots 没有任何引用链相连,就说明此对象不可用

JVM 中可以被当作 GC roots 对象的有以下几种:

    - 栈:
        1. 虚拟机栈中局部变量表中引用的对象
        2. 本地方法栈 jni 引用的对象
    - 方法区
        1. 类中静态变量引用的对象
        2. 常量引用的对象
    - 堆
        1. 存活的 Thread 对象

### java1.2 之后,将引用类型分为以下四种

- 强引用: `Object o=new Object();` 这种属于强引用,这种对象永远不会被回收.
- 软引用: 在发生 oom 之前还会再进行一次 gc,这次 gc 会把软引用对象回收掉,如果内存依然不够,会发生 oom. 实现类`SoftReference`
- 弱引用: 对象可以存活到下次 gc 之前,一旦发生 gc,就会被回收. 实现类 `WeakReference`
- 虚引用: 幽灵引用. 对对象没有任何影响. 在回收的时候,jvm 会收到此对象被回收的通知. 实现类 `PhantomReference`

### 方法区的回收

<JAVA 虚拟机规范> 中不要求虚拟机在方法区实现垃圾收集,也有不支持方法区类型卸载的垃圾回收器,例如 ZGC. 方法区主要回收字符串常量和不再使用的类.

如果字符串常量没在任何地方被引用,就属于废弃常量了.

如果要判定一个类是否属于不再被使用的条件必须符合以下

- 该类所有实例都已被回收
- 加载该类的类加载器已被回收
- 该类对应的`java.lang.Class` 对象没有在任何地方被引用.

## 垃圾收集算法

- **标记-清除(Mark And Sweep):** 从一个跟节点进行扫描,标记出所有存活对象,最后扫描整个内存空间并清除没有被标记的对象(也可以反过来,标记需要清除的对象,扫描并清楚被标记对象)
  - 缺点:会出现大量的空间碎片,回收后的空间是不连续的.给大对象分配时内存时会提前触发 full gc
  - 缺点 2:执行效率不稳定,如果出现大量需要被回收对象,标记清除过程会跟着对象的增长而效率降低
- **标记-复制算法(Mark and Copy):** 将可用内存分为两块,每次只使用一块,当这一块内存使用完,把所有存活对象复制到另一块内存中,再清理掉刚才那块内存.
  - 使用场景:存活对象较少的情况下比较高效,适用于新生代
  - 缺点:可用内存是原来的一半. 对象存活率较高时,效率就会减低,会产生大量的对象复制.
- **标记-整理(Mark-Sweep-Compact):** 从根节点进行扫描,标记出所有存活对象,然后扫描整个空间并清除没有被标记的对象,最后所有对象左移
  - 适用于老年代
  - 缺点是需要移动对象,扫描了整个空间两次(第一次标记存活对象,第二次清除未标记对象)
  - 优点是不会产生空间碎片
- **增量算法**:让垃圾收集线程和应用程序线程交替执行.每次垃圾收集线程只收集一小部分的内存空间,然后切换到应用程序线程.依次反复,直到垃圾收集完成.

## HotSpot 的算法细节实现

要找到清理的对象,首先要进行枚举 GC roots 对象. 如果每次进行 GC Roots 枚举,都需要把整个内存扫描一遍,那会非常浪费时间. 在 HotSpot 的解决方案里,使用一组称为 OopMap 的数据结构来实现直接查找.**OopMap 记录了栈上本地变量到堆上对象的引用关系**.

OopMap 可以帮助 HotSpot 快速完成 GC Roots 扫描. 但是 OopMap 可能会经常发生变化,并且不能频繁更新 OopMap,所以 HotSpot 采用了安全点的方式去更新 OopMap.并且扫描 GC Roots 的时候是 Stop the world 的.程序主要采用两种方法进行停顿的

- 抢先式中断:直接中断所有线程,如果有线程发现不在安全点,则恢复,跑到安全点再停顿.
- 主动式中断:jvm 设置一个中断标识,线程去轮询标识. 一旦发现需要中断,线程跑到最近的安全点就会主动中断自己.

除此安全点之外，还有一个叫做 “安全区域” 的东西，一个一直在执行的线程可以自己 “走” 到安全点去，可是一个处于 Sleep 或者 Blocked 状态的线程是没办法自己到达安全点中断自己的，我们总不能让 GC 操作一直等着这些个 ”不执行“ 的线程重新被分配资源.对于这种情况,需要使用安全区域来解决.

**安全区域是指在一块代码中,引用关系不会改变.因此在这块区域任何地方进行 GC 操作都是安全的.**

当线程执行到安全区域时,首先会把自己标识为 Safe Region. JVM 发起 GC 时,不会理会这个线程.当线程需要离开安全区域时,首先判断当前 JVM 是否完成 GC Roots 的扫描(或者垃圾收集流程中其他需要中断用户线程的阶段),如果安全,会直接离开.否则会一直等待知道可以离开.


## 内存分配和回收策略

大多数情况,新对象优先在 edan 区进行分配. 当 edan 区没有足够空间时,jvm 发起一次 minor gc.

- 大对象直接进入老年代
  大对象是指需要连续的内存空间.例如大的字符串或者元素数量很大的数组.如果大对象进行复制的话,会浪费很多内存空间. 所以 HotSpot 提供了`-XX:PretenureSizeThreshold`参数,指定大对象直接分配在老年代中,避免了在年轻代和 survivor 区之间来回复制.

  _`-XX:PretenureSizeThreshold`参数只对 Serial 和 ParNew 两款新生代回收器有效_

- 长期存活的对象进入老年代
  因为 HotSpot 大部分回收器都采用的分代回收,那内存回收时必须能决策哪些对象存放在新生代,哪些存放在老年代.虚拟机为每个对象设置了一个年龄,当对象在新生代经过一次 minor gc 时,年龄+1.当熬过一定年龄时(默认是 15,可用`-XX:MaxTenuringThreshold=15`配置),就会晋升到老年代中.

- 动态对象年龄判定
  为了能更好的适应不同程序的内存情况,HotSpot 并不是永远要求对象必须到年龄才晋升到老年代.**如果 Survivor 区中所有对象大小总和大于 Survivor 空间的一半,则年龄大于或等于该年龄的对象可以直接进入老年代.**

- 空间分配担保:
  在发生 Minor GC 之前

  1.  虚拟机必须先检查老年代最大可用的连续空间是否大于新生代所有对象总空间,如果这个条件成立,那这一次 Minor GC 可以确保是安全的.
  2.  如果不成立,则虚拟机会先查看`-XX:HandlePromotionFailure` 参数的设置值是否允许担保失败(Handle Promotion Failure);
  3.  如果允许,那会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小,如果大于,将尝试进行一次 Minor GC,尽这次 Minor GC 是有风险的;
  4.  如果小于,或者`-XX：HandlePromotionFailure` 设置不允许冒险,那这时就要改为进行一次 Full GC.

  _**在 JDK 6 Update24 之后,这个`-XX:HandlePromotionFailure`参数就没有用了.此版本之后的规则变为只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小,就会进行 Minor GC,否则进行 Full GC.**_


## 垃圾收集器

> 并发和并行区别:
>
> 1. 并行是指多个垃圾收集线程同时执行,用户线程暂停工作
> 2. 并发是指垃圾收集线程和用户线程同时执行,或者交替执行.
>
> 吞吐量= 执行用户代码时间/(执行用户代码时间+GC 所用时间)

- Serial . 启用参数`-XX:+UseSerialGC`
  新生代收集器,客户端模式下默认收集器.串行(单线程).

- ParNew 收集器[新生代]
  Serial 收集器的多线程版本

- Parallel Scavenge 收集器[新生代]
  特性和 ParNew 类似,侧重点是吞吐量,尽可能的缩短用户线程的停顿时间.

- Serial Old
  老年代,单线程收集器

- Parallel Old 收集器[老年代]:
  Parallel Scavenge 的老年代版本,支持多线程并发收集,使用标记整理算法.

- CMS 收集器[老年代]:
  以获取最短回收停顿时间为目标的收集器,基于"标记-清除"算法实现. **特点是低延迟** 工作流程如下

  1. 初始标记: 标记与 GC Root 直接关联的对象,速度很快,需要"Stop the world"
  2. 并发标记: 扫描与 GC root 直接关联对象的对象图.耗时最长
  3. 重新标记: 修正并发标记期间产生变动的对象,需要"Stop the world"
  4. 并发清除: 清理删除已经判定死亡的对象.

  ![CMS收集器运行示意图](https://raw.githubusercontent.com/Killy412/killy-note/master/img/CMS%E6%94%B6%E9%9B%86%E5%99%A8%E8%BF%90%E8%A1%8C%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg)

- Garbage First (G1) 收集器

  在实现高吞吐量的同时,尽可能地满足垃圾收集暂停时间的要求.G1 把连续的 java 堆内存划分为大小相同的独立区域(Region),每一个 Region 都可以根据需要扮演独立的 Edan/Servivor/老年代空间.回收策略是优先处理回收价值最大的 Region.工作流程如下:

  1. 初始标记: 标记 GC Root 直接关联的对象
  2. 并发标记: 从 GC Root 开始对堆中对象进行可达性分析,递归扫描整个对象图,标记要回收的对象.可与用户线程并发执行.
  3. 最终标记: 对用户线程做一个短暂的暂停.用于处理并发阶段结束后仍遗留下来的少量 SATB 记录.
  4. 筛选回收:负责更新 Region 的统计数据,对各个 Region 的回收成本和价值进行排序,根据用户的期望时间制定回收计划,可以自由选择任意多个 Region 进行回收.然后决定把哪块 Region 复制到空的 Region 中,清理掉旧的 Region 空间.整个操作必须暂停用户线程,由多条收集器并行完成.

  ![G1收集器运行示意图](https://raw.githubusercontent.com/Killy412/killy-note/master/img/G1%E6%94%B6%E9%9B%86%E5%99%A8%E8%BF%90%E8%A1%8C%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg)

- ZGC 收集器

  JDK11 推出的低延迟垃圾收集器,适用于大内存低延迟的内存管理和回收. 可伸缩/低延迟的垃圾收集器. GC 停顿时间不超过 10ms.应用吞吐能力不会下降 15%.

- Shenandoah

  java12 引入.RedHat 开发的. 与 G1 收集器类似,基于 Region 设计的垃圾回收器.停顿时间和堆大小没有关系.停顿时间与 ZGC 接近

### 常用组合

|      Young      |       Old       |                JVM Option                |
| :-------------: | :-------------: | :--------------------------------------: |
|     Serial      |     Serial      |            -XX:+UserSerialGC             |
|    Parallel     | Parallel/Serial | -XX:+UseParallelGC -XX:+UseParallelOldGC |
| Serial/Parallel |       CMS       |   -XX:+UseParNewGC -XX:+UseConcSweepGC   |
|       G1        |        -        |               -XX:+UseG1GC               |




## GC 参数相关

开启 GC 日志

```shell
java -XX:+PrintFlagsFinal -version | grep HeapSize
# 开启内存日志
-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps
```

- 打印详情
  - -XX: -PrintGCDetails
  - -XX:+PrintGCDateStamps
  - -XX:+PrintGCTimeStamps
- jdk8 默认参数
  - -Xmx(1/4 PhyMem) -Xms (1/64 PhyMem[物理内存])
  - Parallel GC 多线程 GC
   - -XX:NewRatio=2 (年轻代和老年代大小比例,2为年轻代:老年代 1:2) -XX: SurvivorRatio=8(设置年轻代中survivor和edan区比例 8为1:8)
  - -XX:+UseAdaptiveSizePolicy
- jvm 常用参数

  ```shell
  -Xms              #设置堆的初始大小. 默认是物理内存的1/64
  -Xmx              #设置堆的最大空间大小. 默认是物理内存的1/4
  -Xmn              #设置年轻代大小
  -Xss              #设置每个线程的堆栈大小
  -XX:NewSize       #设置新生代最小空间大小
  -XX:MaxNewSize    #设置新生代最大空间大小
  -XX:PermSize      #设置永久代最小空间大小(jdk1.8已被弃用)
  -XX:MaxPermSize   #设置永久代最大空间大小(jdk1.8已被弃用)
  -XX:MetaspaceSize #设置元空间初始大小
  -XX:MaxMetaspaceSize #设置元空间最大大小

  -XX:+UseParallelGC  #选择垃圾收集器为并行收集器。此配置仅对年轻代有效。即上述配置下,年轻代使用并发收集,而年老代仍旧使用串行收集。
  -XX:ParallelGCThreads=20  #配置并行收集器的线程数,即:同时多少个线程一起进行垃圾回收。此值最好配置与处理器数目相等。
  -XX:+UseConcMarkSweepGC  #老年代开启CMS GC
  -XX:+UseParNewGC   # 新生代使用并行收集器
  -XX:+HeapDumpOnOutOfMemoryError  # 当oom时把堆内存dump下来

  ```

### 服务端常用组合

```
ENV JAVA_OPTS="\
-server \
-Xmx256m \
-Xms256m \
-Xmn128m \
-XX:SurvivorRatio=8 \
-XX:MetaspaceSize=64m \
-XX:MaxMetaspaceSize=128m \
-XX:+UseParallelGC \
-XX:ParallelGCThreads=4 \
-XX:+UseParallelOldGC \
-XX:+UseAdaptiveSizePolicy \
-XX:+PrintGCDetails \
-XX:+PrintTenuringDistribution \
-XX:+PrintGCTimeStamps \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/ \
-Xloggc:/gc.log \
-XX:+UseGCLogFileRotation \
-XX:NumberOfGCLogFiles=5 \
-XX:GCLogFileSize=10M"
```
