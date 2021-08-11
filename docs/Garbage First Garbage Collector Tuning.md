# Garbage First Garbage Collector Tuning

*by Monica Beckwith*

*Yezhou 译*

Published August 2013

2021年08月09日创建

2021年08月11日修改

[原文链接](https://www.oracle.com/technical-resources/articles/java/g1gc.html)

**Learn about how to adapt and tune the G1 GC for evaluation, analysis and performance.**

**学习配置、调优G1 GC时如何评估分析性能**

> The [Garbage First Garbage Collector (G1 GC)](https://www.oracle.com/java/technologies/javase/hotspot-garbage-collection.html) is the low-pause, server-style generational garbage collector for Java HotSpot VM. The G1 GC uses concurrent and parallel phases to achieve its target pause time and to maintain good throughput. When G1 GC determines that a garbage collection is necessary, it collects the regions with the least live data first (garbage first).

[Garbage First Garbage Collector (G1 GC)](https://www.oracle.com/java/technologies/javase/hotspot-garbage-collection.html)是一款HotSpot虚拟机内置的分代垃圾回收器，有低延迟、面向服务端设计的特点。它使用了一些并发、并行的阶段，来达成目标停顿时间和良好吞吐量的目标。G1进行垃圾回收时，会优先收集拥有最少存活对象的区域（垃圾优先）。

> A garbage collector (GC) is a memory management tool. The G1 GC achieves automatic memory management through the following operations:
> - Allocating objects to a young generation and promoting aged objects into an old generation.
> - Finding live objects in the old generation through a concurrent (parallel) marking phase. The Java HotSpot VM triggers the marking phase when the total Java heap occupancy exceeds the default threshold.
> - Recovering free memory by compacting live objects through parallel copying.

垃圾回收器是一种内存管理工具。G1通过以下操作完成自动内存管理：

- 在年轻代分配对象，将年老对象升入老年代。
- 在老年代寻找存活对象时会采用并发标记。一旦堆空间占用超过阈值，HotSpot虚拟机会触发这个行为。
- 释放内存空间时，会并行地复制存活对象后压缩整理。

> Here, we look at how to adapt and tune the G1 GC for evaluation, analysis and performance—we assume a basic understanding of Java garbage collection.

接下来，让我们学习配置、调优G1 GC时如何评估分析性能——假定读者了解基础的Java垃圾回收机制。

>  The G1 GC is a regionalized and generational garbage collector, which means that the Java object heap (heap) is divided into a number of equally sized regions. Upon startup, the Java Virtual Machine (JVM) sets the region size. The region sizes can vary from 1 MB to 32 MB depending on the heap size. The goal is to have no more than 2048 regions. The eden, survivor, and old generations are logical sets of these regions and are not contiguous.

G1是分区分代的垃圾回收器，会将Java堆分割成多个大小相同的区域。JVM启动时会根据堆大小设置区域大小，从1MB到32MB不等，使最终区域数不超过2048个。eden区、survivor区、老年代是部分区域的逻辑集合，且这些区域不要求连续。

> The G1 GC has a pause time-target that it tries to meet (soft real time). During young collections, the G1 GC adjusts its young generation (eden and survivor sizes) to meet the soft real-time target. During mixed collections, the G1 GC adjusts the number of old regions that are collected based on a target number of mixed garbage collections, the percentage of live objects in each region of the heap, and the overall acceptable heap waste percentage.

G1有一个目标停顿时间（软实时）。进行年轻代回收时，G1会调整年轻代（eden和survivor区的大小）来适应软实时需求。进行混合回收时，G1会根据目标回收数量、区域内存活对象占比和可接受的堆内存浪费空间占比，调整需要回收的老年代区域数量。

> The G1 GC reduces heap fragmentation by incremental parallel copying of live objects from one or more sets of regions (called Collection Set (CSet)) into different new region(s) to achieve compaction. The goal is to reclaim as much heap space as possible, starting with those regions that contain the most reclaimable space, while attempting to not exceed the pause time goal (garbage first).

G1的压缩操作会从多个区域集合（称为Collection Set，CSet）增量式并行地复制存活对象到新区域，以减少堆内存碎片。为了尽可能多地回收堆内存空间，G1往往从可回收空间最多的区域开始收集，兼顾不超过目标停顿时间的需求（垃圾优先）。

> The G1 GC uses independent Remembered Sets (RSets) to track references into regions. Independent RSets enable parallel and independent collection of regions because only a region's RSet must be scanned for references into that region, instead of the whole heap. The G1 GC uses a post-write barrier to record changes to the heap and update the RSets.

G1使用Remembered Sets（RSets）来追踪某个区域内的引用。独立的RSet使每个区域都可以独立且并行地回收垃圾，因为回收时只需扫描某个区域的RSet来检查该区域的引用，而不用扫描整个堆。G1使用写后屏障来记录堆的变化并更新RSets。

#### Garbage Collection Phases

> Apart from evacuation pauses (described below) that compose the stop-the-world (STW) young and mixed garbage collections, the G1 GC also has parallel, concurrent, and multiphase marking cycles. G1 GC uses the Snapshot-At-The-Beginning (SATB) algorithm, which takes a snapshot of the set of live objects in the heap at the start of a marking cycle. The set of live objects is composed of the live objects in the snapshot, and the objects allocated since the start of the marking cycle. The G1 GC marking algorithm uses a pre-write barrier to record and mark objects that are part of the logical snapshot.

除了年轻代回收和混合回收这两个需要STW的迁移停顿过程（下文描述），G1还有可分为多个阶段的标记操作过程，部分阶段是并行、并发的。G1使用原始快照（SATB）算法记录了堆中所有在标记操作开始时存活的对象。存活对象来自两个部分：在快照中存活的对象和在标记开始后新生成的对象。标记算法采用写前屏障来标记逻辑上属于快照的对象。

#### Young Garbage Collections

> The G1 GC satisfies most allocation requests from regions added to the eden set of regions. During a young garbage collection, the G1 GC collects both the eden regions and the survivor regions from the previous garbage collection. The live objects from the eden and survivor regions are copied, or evacuated, to a new set of regions. The destination region for a particular object depends upon the object's age; an object that has aged sufficiently evacuates to an old generation region (that is, promoted); otherwise, the object evacuates to a survivor region and will be included in the CSet of the next young or mixed garbage collection.

G1会在eden区域集合处理大多数内存分配请求。年轻代回收时，G1会收集eden区域和上一次回收的survivor区域，存活的对象会被迁移到新的区域集合，具体去哪里取决于对象的年龄，足够老的对象会被迁移到老年代区域，否则对象会进入survivor区域，这个区域也会在下一次年轻代或者混合垃圾回收的CSet之内。

#### Mixed Garbage Collections

> Upon successful completion of a concurrent marking cycle, the G1 GC switches from performing young garbage collections to performing mixed garbage collections. In a mixed garbage collection, the G1 GC optionally adds some old regions to the set of eden and survivor regions that will be collected. The exact number of old regions added is controlled by a number of flags that will be discussed later (see "[Taming Mixed GCs](https://www.oracle.com/technical-resources/articles/java/g1gc.html#Taming)"). After the G1 GC collects a sufficient number of old regions (over multiple mixed garbage collections), G1 reverts to performing young garbage collections until the next marking cycle completes.

并发标记过程成功后，G1从年轻代垃圾回收转换到混合垃圾回收。混合垃圾回收时，除了原本的eden和survivor区域，G1会选择性加入一些老年代区域进行收集。加入的老年代区域数量受到之后讨论的一些标识控制（见“[Taming Mixed GCs](https://www.oracle.com/technical-resources/articles/java/g1gc.html#Taming)“）。回收必要数量的老年代区域后（进行多次混合垃圾回收后），G1会回到年轻代垃圾回收的状态，直到下一次标记过程完成。

#### Phases of the Marking Cycle

> The marking cycle has the following phases:
> - **Initial mark phase**: The G1 GC marks the roots during this phase. This phase is piggybacked on a normal (STW) young garbage collection.
> - **Root region scanning phase**: The G1 GC scans survivor regions of the initial mark for references to the old generation and marks the referenced objects. This phase runs concurrently with the application (not STW) and must complete before the next STW young garbage collection can start.
> - **Concurrent marking phase**: The G1 GC finds reachable (live) objects across the entire heap. This phase happens concurrently with the application, and can be interrupted by STW young garbage collections.
> - **Remark phase**: This phase is STW collection and helps the completion of the marking cycle. G1 GC drains SATB buffers, traces unvisited live objects, and performs reference processing.
> - **Cleanup phase**: In this final phase, the G1 GC performs the STW operations of accounting and RSet scrubbing. During accounting, the G1 GC identifies completely free regions and mixed garbage collection candidates. The cleanup phase is partly concurrent when it resets and returns the empty regions to the free list.

完整的标记过程有以下几个阶段：

- **初始标记阶段**：G1会标记GC根直达的对象。这个阶段依赖一次STW的年轻代垃圾回收。
- **根区域扫描阶段**：G1扫描上阶段标记的survivor区域中的指向老年代区域的引用，标记相应的对象。这个阶段会和应用并发执行（不进行STW）且这个阶段必须完成，才能开始下一次STW的年轻代垃圾回收。
- **并发标记阶段**：G1在堆上寻找所有可达（存活）对象。这个阶段和应用并发执行，且可以被STW的年轻代垃圾回收中断。
- **重新标记阶段**：这个阶段会STW来完成标记。G1使用SATB算法产生的记录，跟踪未访问的存活对象，处理引用。
- **清理阶段**：最后阶段，G1会STW来审计和刷洗RSet。审计时，G1会识别完全空闲的区域和混合垃圾回收的候选区域。清理时，重置并返回空闲区域到空闲列表的操作是并发的。

#### Important Defaults

> The G1 GC is an adaptive garbage collector with defaults that enable it to work efficiently without modification. Here is a list of important options and their default values. This list applies to the latest Java HotSpot VM, build 24. You can adapt and tune the G1 GC to your application performance needs by entering the following options with changed settings on the JVM command line.

G1是一款自适应的垃圾回收器，在默认的配置下已可获得良好的性能。下面是一些重要选项和它们的默认值，这些项已经应用到最新的HotSpot虚拟机build 24中。你可以使用JVM命令行配置调试这些设置来满足性能需求。

> - `-XX:G1HeapRegionSize=n`
>
> Sets the size of a G1 region. The value will be a power of two and can range from 1MB to 32MB. The goal is to have around 2048 regions based on the minimum Java heap size.

设置区域的大小，值为2整数倍，范围从1MB到32MB。目标是在堆大小尽可能小的情况下控制区域的数量在2048个左右。

> - `-XX:MaxGCPauseMillis=200`
>
> Sets a target value for desired maximum pause time. The default value is 200 milliseconds. The specified value does not adapt to your heap size.

设置最大停顿时间，默认为200毫秒，该值并没有适配你堆的大小。

> - `-XX:G1NewSizePercent=5`
> 
> Sets the percentage of the heap to use as the minimum for the young generation size. The default value is 5 percent of your Java heap. This is an experimental flag. See "[How to unlock experimental VM flags](#How-to-Unlock-Experimental-VM-Flags)" for an example. This setting replaces the `-XX:DefaultMinNewGenPercent` setting. This setting is not available in Java HotSpot VM, build 23.

设置年轻代占堆的最小百分比，默认为5%。这是一个实验性标志，查看"[How to unlock experimental VM flags](#How-to-Unlock-Experimental-VM-Flags)"获取示例。这个设置代替了`-XX:DefaultMinNewGenPercent`，在HotSpot虚拟机build 23或更早的版本上不可用

> - `-XX:G1MaxNewSizePercent=60`
>
> Sets the percentage of the heap size to use as the maximum for young generation size. The default value is 60 percent of your Java heap. This is an experimental flag. See "[How to unlock experimental VM flags](#How-to-Unlock-Experimental-VM-Flags)" for an example. This setting replaces the `-XX:DefaultMaxNewGenPercent` setting. This setting is not available in Java HotSpot VM, build 23.

设置年轻代占堆的最大百分比，默认为60%。这是一个实验性标志，查看"[How to unlock experimental VM flags](#How-to-Unlock-Experimental-VM-Flags)"获取示例。这个设置代替了`-XX:DefaultMaxNewGenPercent`，在HotSpot虚拟机build 23或更早的版本上不可用

> - `-XX:ParallelGCThreads=n`
>
> Sets the value of the STW worker threads. Sets the value of n to the number of logical processors. The value of `n` is the same as the number of logical processors up to a value of 8.

设置STW时的工作线程数量，建议设置成逻辑处理器的数量，直到为8

> If there are more than eight logical processors, sets the value of `n` to approximately 5/8 of the logical processors. This works in most cases except for larger SPARC systems where the value of `n` can be approximately 5/16 of the logical processors.

如果逻辑处理器的数量大于8，设置成5/8的逻辑处理器数量左右，就能适应大多数需求。若使用的是大型SPARC系统，建议设置成5/16的逻辑处理器数量。

> - `-XX:ConcGCThreads=n`
>
> Sets the number of parallel marking threads. Sets `n` to approximately 1/4 of the number of parallel garbage collection threads (`ParallelGCThreads`).

设置并行标记线程的数量，建议设置成1/4的并行垃圾回收线程数量（即上一个ParallelGCThreads）

> - `-XX:InitiatingHeapOccupancyPercent=45`
>
> Sets the Java heap occupancy threshold that triggers a marking cycle. The default occupancy is 45 percent of the entire Java heap.

设置开始标记过程的堆的占用百分比阈值，默认为整个堆大小的45%。

> - `-XX:G1MixedGCLiveThresholdPercent=65`
>
> Sets the occupancy threshold for an old region to be included in a mixed garbage collection cycle. The default occupancy is 65 percent. This is an experimental flag. See "[How to unlock experimental VM flags](#How-to-Unlock-Experimental-VM-Flags)" for an example. This setting replaces the `-XX:G1OldCSetRegionLiveThresholdPercent` setting. This setting is not available in Java HotSpot VM, build 23.

设置混合垃圾回收时，进行回收的老年代区域需要满足的内存占用百分比阈值，默认为65%。这是一个实验性标志，查看"[How to unlock experimental VM flags](#How-to-Unlock-Experimental-VM-Flags)"获取示例。这个设置代替了`-XX:G1OldCSetRegionLiveThresholdPercent`，在HotSpot虚拟机build 23或更早的版本上不可用。

> - `-XX:G1HeapWastePercent=10`
>
> Sets the percentage of heap that you are willing to waste. The Java HotSpot VM does not initiate the mixed garbage collection cycle when the reclaimable percentage is less than the heap waste percentage. The default is 10 percent. This setting is not available in Java HotSpot VM, build 23.

设置堆的浪费百分比阈值。如果可回收的空间占比小于这个值，HotSpot不会启动混合垃圾回收。默认为10%，这个设置在HotSpot虚拟机build 23或更早的版本不可用。

> - `-XX:G1MixedGCCountTarget=8`
>
> Sets the target number of mixed garbage collections after a marking cycle to collect old regions with at most `G1MixedGCLIveThresholdPercent` live data. The default is 8 mixed garbage collections. The goal for mixed collections is to be within this target number. This setting is not available in Java HotSpot VM, build 23.

设置在标记完成后回收达到占用阈值的老年代区域的混合垃圾回收次数。默认是8次，真实次数会在该值之内，这个设置在HotSpot虚拟机build 23或更早的版本上不可用。

> - `-XX:G1OldCSetRegionThresholdPercent=10`
>
> Sets an upper limit on the number of old regions to be collected during a mixed garbage collection cycle. The default is 10 percent of the Java heap. This setting is not available in Java HotSpot VM, build 23.

设置在一次混合垃圾回收过程中，老年代区域回收数量占整个Java堆区域比例的最大值。默认为10%，这个设置在HotSpot虚拟机build 23或更早的版本上不可用。

> - `-XX:G1ReservePercent=10`
>
> Sets the percentage of reserve memory to keep free so as to reduce the risk of to-space overflows. The default is 10 percent. When you increase or decrease the percentage, make sure to adjust the total Java heap by the same amount. This setting is not available in Java HotSpot VM, build 23.

设置始终保持空闲的保留内存占堆的比值，这部分内存是为了防范存活对象没有足够内存来迁移的风险，默认为10%。当你更改这个设置时，记得调整Java堆的大小。这个设置在HotSpot虚拟机build 23或更早的版本上不可用。

#### How to Unlock Experimental VM Flags
> To change the value of experimental flags, you must unlock them first. You can do this by setting -XX:+UnlockExperimentalVMOptions explicitly on the command line before any experimental flags. For example:

要改变实验性标识的值，需要先解锁。你需要在命令行的所有实验性标识之前，显式设置-XX:+UnlockExperimentalVMOptions，比如：

> java -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=10 -XX:G1MaxNewSizePercent=75 G1test.jar

#### Recommendations

> When you evaluate and fine-tune G1 GC, keep the following recommendations in mind:

当你在评估和调优G1时，请记住以下几点：

> - **Young Generation Size**: Avoid explicitly setting young generation size with the `-Xmn` option or any or other related option such as `-XX:NewRatio`. Fixing the size of the young generation overrides the target pause-time goal.

- **年轻代大小**：不要用```-Xmn```或者其他相关的选项如```-XX:NewRatio```来显式设置年轻代的大小。固定年轻代的大小会导致目标停顿时间被无视。

> - **Pause Time Goals**: When you evaluate or tune any garbage collection, there is always a latency versus throughput trade-off. The G1 GC is an incremental garbage collector with uniform pauses, but also more overhead on the application threads. The throughput goal for the G1 GC is 90 percent application time and 10 percent garbage collection time. When you compare this to Java HotSpot VM's throughput collector, the goal there is 99 percent application time and 1 percent garbage collection time. Therefore, when you evaluate the G1 GC for throughput, relax your pause-time target. Setting too aggressive a goal indicates that you are willing to bear an increase in garbage collection overhead, which has a direct impact on throughput. When you evaluate the G1 GC for latency, you set your desired (soft) real-time goal, and the G1 GC will try to meet it. As a side effect, throughput may suffer.

- **目标停顿时间**：无论你如何评估和调整任意垃圾回收器，都要权衡延迟和吞吐量。G1是一款存在一致停顿的增量式垃圾回收器，对应用程序的线程开销也更大。G1的目标吞吐量为90%的应用时间和10%的垃圾回收时间。与之相比，HotSpot虚拟机的吞吐量优先垃圾回收器则是99%的应用时间和1%的垃圾回收时间。因此，评估G1的吞吐量时，请放宽你的目标停顿时间。如果你使用激进的设置，则必然要接受因此增加的垃圾回收的开销。这也会直接影响吞吐量。评估G1的延迟时，设置一个你想要的软实时目标，G1会尝试达成这个目标，同时吞吐量也可能受此影响。

> - **Taming Mixed Garbage Collections**: Experiment with the following options when you tune mixed garbage collections. See "[Important Defaults](#Important-Defaults)" for information about these options:
>   - `-XX:InitiatingHeapOccupancyPercent`
>     For changing the marking threshold.
>   - `-XX:G1MixedGCLiveThresholdPercent` and `-XX:G1HeapWastePercent`
>     When you want to change the mixed garbage collections decisions.
>   - `-XX:G1MixedGCCountTarget` and `-XX:G1OldCSetRegionThresholdPercent`
>     When you want to adjust the CSet for old regions.

- **控制混合垃圾回收**：当你调试混合垃圾回收时，尝试下述选项。查看"[Important Defaults](#Important-Defaults)"获取关于这些选项的更多信息：

  - `-XX:InitiatingHeapOccupancyPercent`

    改变标记的阈值

  - `-XX:G1MixedGCLiveThresholdPercent`和`-XX:G1HeapWastePercent`

    当你要想更改混合垃圾回收的决策时

  - `-XX:G1MixedGCCountTarget`和`-XX:G1OldCSetRegionThresholdPercent`

    当你想要调整CSet中的老年区域数时

#### Overflow and Exhausted Log Messages

> When you see to-space overflow/exhausted messages in your logs, the G1 GC does not have enough memory for either survivor or promoted objects, or for both. The Java heap cannot expand since it is already at its max. Example messages:
> `924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space exhausted), 0.1957310 secs]`
> OR
> `924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space overflow), 0.1957310 secs]`

如果你在GC日志中发现了to-space overflow/exhausted消息，说明G1已经没有足够的空间来迁移存活的对象或晋升的老年对象了。堆的大小已经达到了极限。示例消息：

`924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space exhausted), 0.1957310 secs]`
或
`924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space overflow), 0.1957310 secs]`

> To alleviate the problem, try the following adjustments:

为了缓解这个问题，可以尝试以下调整：

> Increase the value of the `-XX:G1ReservePercent` option (and the total heap accordingly) to increase the amount of reserve memory for "to-space".
> Start the marking cycle earlier by reducing the `-XX:InitiatingHeapOccupancyPercent`.
> You can also increase the value of the `-XX:ConcGCThreads` option to increase the number of parallel marking threads.
> See "[Important Defaults](#Important-Defaults)" for a description of these options.

增加`-XX:G1ReservePercent`选项的值（相应也调节整个堆的大小）来增加为"to-space"保留的内存。

通过减少`-XX:InitiatingHeapOccupancyPercent`的值，更早地开始标记过程。

你也可以增加`-XX:ConcGCThreads`选项的值来增加并发标记线程的数量。

查看"[Important Defaults](#Important-Defaults)"来获取这些选项的具体描述。

#### Humongous Objects and Humongous Allocations

> For G1 GC, any object that is more than half a region size is considered a "Humongous object". Such an object is allocated directly in the old generation into "Humongous regions". These Humongous regions are a contiguous set of regions. `StartsHumongous` marks the start of the contiguous set and `ContinuesHumongous` marks the continuation of the set.

G1认为所有超过区域大小一般的对象是“巨大对象”。这种对象会直接进入老年代并在“巨型区域”中分配。巨型区域是一段连续相邻区域的集合，`StartsHumongous`标记了这段连续区域的开始，`ContinuesHumongous`标记了当前区域是否与上个区域构成同一个巨型区域。

> Before allocating any Humongous region, the marking threshold is checked, initiating a concurrent cycle, if necessary.

在分配任何巨型区域之前，标记阈值会被检查，初始化一个标记过程，如果必要。

> Dead Humongous objects are freed at the end of the marking cycle during the cleanup phase also during a full garbage collection cycle.

死亡的巨大对象会在标记过程结束时的清理阶段被清除，或者也会在全量垃圾回收过程中被清除。

> In-order to reduce copying overhead, the Humongous objects are not included in any evacuation pause. A full garbage collection cycle compacts Humongous objects in place.

为了减少复制的开销，巨大对象不会参与任何复制停顿操作。一个全量垃圾回收过程直接在原地压缩巨大对象。

> Since each individual set of StartsHumongous and ContinuesHumongous regions contains just one humongous object, the space between the end of the humongous object and the end of the last region spanned by the object is unused. For objects that are just slightly larger than a multiple of the heap region size, this unused space can cause the heap to become fragmented.

因为巨型区域的每个独立的区域（StartsHumongous或ContinuesHumongous区域）仅包含一个巨大对象，那么这个从对象的尾部到这个对象横跨的最后一个区域的尾部的空间是不使用的。所以那些只比堆区域大小的整数倍大一点点的对象产生的未使用空间会造成堆的碎片化。

> If you see back-to-back concurrent cycles initiated due to Humongous allocations and if such allocations are fragmenting your old generation, please increase your `-XX:G1HeapRegionSize` such that previous Humongous objects are no longer Humongous and will follow the regular allocation path

如果你发现因为巨大对象分配产生的老年代碎片使得并发的内存回收过程一次又一次发生，请增加`-XX:G1HeapRegionSize`的值使得之前被认为是巨大对象的对象回归正常的分配规则。

#### Conclusion

> G1 GC is a regionalized, parallel-concurrent, incremental garbage collector that provides more predictable pauses compared to other HotSpot GCs. The incremental nature lets G1 GC work with larger heaps and still provide reasonable worst-case response times. The adaptive nature of G1 GC just needs a maximum soft-real time pause-time goal along-with the desired maximum and minimum size for the Java heap on the JVM command line.

G1是一款分区的，并行并发的，增量式的垃圾回收器，它对比其他HotSpot的垃圾回收器提供了更加可预测的停顿。增量式的特性使得G1在面对更大的堆时仍然保持了合理的最坏响应时间。自适应的特性使得G1在少量配置下能够良好地工作，仅需在JVM命令行上配置一个软实时时间的最大停顿目标时间和理想的Java堆空间大小即可。

#### See Also

> - [The Garbage-First Garbage Collector](https://www.oracle.com/java/technologies/javase/hotspot-garbage-collection.html)
> - [Java HotSpot Garbage Collection](https://www.oracle.com/java/technologies/javase/javase-core-technologies-apis.html)

#### About the Author

> Monica Beckwith, Principal Member of Technical Staff at Oracle, is the performance lead for the Java HotSpot VM's Garbage First Garbage Collector. She has worked in the performance and architecture industry for over 10 years. Prior to Oracle and Sun Microsystems, Monica lead the performance effort at Spansion Inc. Monica has worked with many industry standard Java based benchmarks with a constant goal of finding opportunities for improvement in the Java HotSpot VM.

译者 Yezhou

#### Join the Conversation

> Join the Java community conversation on [Facebook](https://www.facebook.com/ilovejava), [Twitter](https://twitter.com/#!/java), and the [Oracle Java Blog](https://blogs.oracle.com/java/)!

