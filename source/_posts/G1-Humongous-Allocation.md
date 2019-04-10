title: G1-Humongous Allocation
author: Xiang Chuang
tags:
  - GC-G1
  - JVM
  - GC
categories:
  - 爱学爱问
date: 2019-04-10 11:56:00
---
它是由'G1 Humongous Allocation'造成的。大型对象（Humongous ）是大于G1中region大小50％的对象。频繁大型对象分配会导致性能问题。如果region里面包含大量的大型对象，则该region中最后一个具有巨型对象的区域与区域末端之间的空间将不会使用。如果有多个这样的大型对象，这个未使用的空间可能导致堆碎片化。直到jdk1.8u40之前，这些巨型对象的回收只在full GC期间完成。在较新的JVM中，对这些对象的清理放在了清理阶段。

解决方案：

①.可以加大region的大小，设置-XX:G1HeapRegionSize=n，但是这个参数需要设置为2的幂次方，最小值是1M，最大值是32M。

②.如果可以的话增加JVM堆大小（即-Xmx -Xms）。