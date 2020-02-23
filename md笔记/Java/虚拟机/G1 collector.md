# G1 collector

G1 把堆划分成多个大小相等的**独立区域（Region）**，新生代和老年代不再物理隔离。 

![img](https://camo.githubusercontent.com/5049da1b34969b272be2bffc6c6de0206b33253c/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39626264646565622d653933392d343166302d386538652d3262316130616137653061372e706e67)

可以设置最大的 STW 停顿时间

* -XX:MaxGCPauseMillis=N
* 250 by default

年轻代 GC 算法

* STW，Parallel，Copying

老年代 GC 算法

* Mostly-concurrent marking （vs CMS）
* Incremental compaction

## G1 的内存分布

* 将堆分成若干个等大的区域
* -XX:G1HeapRegionSize= N 2048 by default

## G1 内部细节

* 无需回收整个堆，而是选择一个 Collection Set
* 两种 GC:
  * Fully young GC
  * Mixed GC  （包含一次Fully young GC）

* 估计每个 Region 中的垃圾比例，优先回收垃圾多的 Region
  * That‘s why it’s called Garbage First

## Card Table & Remembered Set

### Card Table

* 表中的每个 entry 覆盖 512 Byte 的内存空间
* 当对应的内存空间发生改变时，标记为 dirty

### Remembered Set

* 指向 Card Table 中的对应 entry
* 可找到具体内存区域

### 时间换空间

* 用额外的空间维护引用信息
* 5%~10% memory overhead

**情景**

有两个 Region1 和 Region2

Region1 中某一个 Card 引用了 Region2 中的某一个 Card，就会在 Region2 中的 RSet 表中写入 Region1 中的卡的地址，避免整个堆的扫描

## Writer Barrier

JVM 注入的一小段代码，用于记录指针变化

例如如下给 field 赋值

```java
object.file = <reference> (putfield)
```

更新指针时

* 标记 Card 为 Dirty
* 将 Card 存入 Dirty Card Queue  有 **白绿黄红** 四个颜色
* 如上个例子中的 Region1 中的某一个 Card

### 更新 Remembered Set

* White 时 什么都不发生
* Green zone （-XX:G1ConcRefineMentGreenZone=N）
  * Refinement 线程开始被激活，开始更新RS

* Yellow zone （-XX:G1ConcRefineMentYellowZone=N）
  * 全部 Refinement 线程开始被激活

* Redzone （-XX:G1ConcRefineMentRedZone=N）
  * 应用线程也参与排空队列的工作

**Refinement 线程是把 Dirty Card Queue 中的 dirty card 放入 Remembered Set**

## Fully Young GC

### STW（Evacuation Pause）

疏散暂停 类似于 compact

1. 构建 CS（Eden + Survivor）

2. 扫描 GC Roots
3. Update RS：排空 Dirty Card Queue
4. Process RS：找到被哪些老年代对象所引用
5. Object Copy
6. Reference Processing

G1 记录每个阶段的时间，用于自动调优

记录 Eden/Survivor 的数量和 GC 时间

* 根据暂停目标自动调整 Region 的数量
* 暂停目标越短，Eden 数量越少  (GC 时间变短，但是变得频繁)（吞吐量下降）
* -XX:+PrintAdaptiveSizePolicy
* -XX:+PrintTenuringDistribution

## Old GC

当堆用量到达一定程度时触发

* -XX:InitiatingHeapOccupancyPercent=N
* 45 by default

Old GC 是并发进行的

* 三色标记算法：不暂停应用线程的情况下进行标记

* 根节点为黑色
* 黑色指向的所有对象都是灰色，灰色对象全部放入一个队列
* 拿出一个灰色对象，把它标记为黑色，并把它指向的对象标记为灰色放入队列

（类似广度优先）

最后情况非黑即白

并发情况下出现一个问题 Lost Object Problem

```java
//有三个对象ABC
//AB为灰 C为白 B引用C
//标记线程：A没有引用C  A变黑 C不变
//应用线程：A引用C、B放弃C
A.c=B.c;
B.c=null;
//标记B黑、C不变误判C为死对象
```

这个时候通过Write Barrier 注入一小段代码在`B.c=null;`这一步仍然记录C为活对象

* Snapshot-At-The-Beginning
  * 保持在remarking 阶段开始的 object graph
  * 可能产生浮动垃圾

### 流程

* STW（Fully young GC）
  * piggy-backed on young GC
  * 利用 Young GC 的信息

* 恢复应用线程
* 并发初始标记（Init marking）三色标记
* STW
  * Remark
  * Cleanup 立即回收全空的区
* 恢复应用线程

## Mixed GC

* 不一定立即发生
* 选择若干个Region进行
  * 默认1/8的 old Region
    * -XX：G1MixedGCCountTarget=N
  * Eden+Survivor Region
  * STW，Parallel，Copying
* 根据暂停目标，选择垃圾最多的 Old Region 优先进行
  * -XX: G1MixedGCLiveThresholdPercent=N (default 85)
  * -XX: G1HeapWastePercent=N

## ZGC/Shenandoah

