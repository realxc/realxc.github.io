title: G1-Evacuation Failure
author: Xiang Chuang
tags:
  - GC-G1
  - JVM
  - GC
categories:
  - 爱学爱问
date: 2019-04-10 11:56:00
---
当没有更多的空闲region被提升到老一代或者复制到幸存空间时，并且由于堆已经达到最大值，堆不能扩展，从而发生Evacuation Failure。对于G1 GC，它是非常耗时的。 

 a.对于成功复制的对象，G1需要更新引用，并且该region被一直引用。

 b.对于未成功复制的对象，G1将自动转发它们，并保留这些region。

解决方案：

①.不要过度加一些jvm参数。比如-Xmn,这个参数会限制G1的参数的自动扩展。可以仅使用-Xms，-Xmx和暂停时间目标-XX：MaxGCPauseMillis，删除任何额外的堆大小，例如-Xmn，-XX：NewSize，-XX：MaxNewSize，-XX：SurvivorRatio等。

②.如果问题仍然存在，则增加JVM堆大小（即-Xmx）。

③.如果您无法增加堆大小，并且您注意到marking cycle没有足够早地开始回收老一代，那么请减少-XX：InitiatingHeapOccupancyPercent。默认值是45％。减小该值将提前开始marking cycle 。另一方面，如果marking cycle 提前开始并且未收回，请将-XX：InitiatingHeapOccupancyPercent阈值增加到默认值以上。

④.如果并发marking cycle准时开始，但需要很长时间才能完成，那么使用属性'-XX：ConcGCThreads'增加并发标记线程数的数量。默认是GC Workers: 1 ，单线程执行。

⑤.如果有大量“空间耗尽（to-space exhausted）”或“空间溢出（to-space overflow）”GC事件，则增加-XX:G1ReservePercent。默认值是Java堆的10％。注意：G1 GC将此值限制在50％以内。