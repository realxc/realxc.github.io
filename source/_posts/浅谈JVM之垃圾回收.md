title: java对象的诞生与消亡
author: Xiang Chuang
tags:
  - JVM
  - 内存调优
  - java对象
  - 垃圾回收
categories:
  - 专题研讨
date: 2019-04-08 16:54:00
---
## 说在前面
&emsp;&emsp;本文将从java对象的角度出发，探讨java应用运行过程中“对象”从诞生到消亡的生命历程，结合jvm的相关技术方案简要阐述各个环节的实现细节。需要指出的是，本文旨在阐述清楚“对象”的生命周期，对于jvm（本文基于HotSpot虚拟机）相关技术的深层次原理只会点到为止（能描述清“对象”的生命周期即可），后续将会在其他文章中单独罗列出各个主题进行深入剖析。

## 概述
+ [java内存区域](#partI)
+ [java对象](#partII)
+ [垃圾回收](#partIII)
+ [内存调优](#partIV)

----------------------------------

## java内存区域
&emsp;&emsp;java对象生存的载体-运行时数据区，虚拟机中数据区大致包含如下图所示的几个区域： 
![upload successful](\images\pasted-74.png)
1、程序计数器
&emsp;&emsp;当前线程所执行的字节码的行号指示器，标识执行到的节点，这是一块较小的内存空间。由于任何时刻，一个处理器（多核处理器中的某一核）只会执行一个线程中的指令，因此，每个线程都有一个独立、私有的程序计数器。此区域（唯一一个）在java虚拟机规范中没有规定任何OOM的情况。
2、虚拟机栈
&emsp;&emsp;其生命周期与线程相同，方法在执行时都会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息，每一个方法的调用就是相应栈帧在虚拟机栈中入栈和出栈的过程。
局部变量表：存放编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（指向对象起始地址的引用指针-HotSpot方案、或者指向一个代表对象的句柄或其他与此对象相关的位置）、returnAddress类型（执行一条字节码指令的地址）。其中，除long和double占用2个局部变量空间（Slot）外，其余只占用1个Slot。局部变量表所需内存空间在编译期间完成分配，运行期间不会改变大小，即，进入一个方法时，栈帧中所需分配的局部变量空间是确定的。java虚拟机规范对此区域规定了两类异常情况，1是线程请求的栈深度大于虚拟机所允许的深度时抛出StackOverflowError，2是可动态扩展的虚拟机栈扩展时无法申请到足够的内存时抛出OOM。
3、本地方法栈
&emsp;&emsp;HotSpot将其与虚拟机栈合二为一。它是为虚拟机所使用到的Native方法服务。它可能抛出的异常与虚拟机栈一致。
4、堆
&emsp;&emsp;此区域用于存放对象的实例。java虚拟机规范原文：The heap is the runtime data area from which memory for all class instances and arrays is allocated.此区域会抛出OOM异常。由于这个区域存分的是对象实例且是线程共享区域，因此在后续的内容中将以此为核心进行讲述。
5、方法区（HotSpot，jdk8之后已由元空间取代）
&emsp;&emsp;该区域用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据，为了与java堆区分开来，java虚拟机规范为其命了一个别名Non-Heap。该区域会抛出OOM。
6、运行时常量池
&emsp;&emsp;方法区的一部分，Clas文件中除了由类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于存放编译期生产的各种字面量和符号引用。除了编译期间会产生常量，java允许运行期间产生常量并存放到运行时常量池中，如String.intern()。该区域是方法区的一部分，因此申请不到内存时也会抛出OOM。
7、直接内存
&emsp;&emsp;它不属于虚拟机运行时数据区的一部分，但其也可能出现OOM，且使用频繁（如NIO类，引入了基于通道（Channel）与缓冲区（Buffer）的I/O方式，可使用Native函数直接分配堆外内存，然后通过一个存储在java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，这样避免了在java堆与Native堆中来回复制数据从而提升了性能）。该区域使用超过机器内存等限制时会抛出OOM。
8、元空间
&emsp;&emsp;如示意图中所述，HotSpot在jdk8（含）之后，已经移除了老年代（方法区），转由元空间（metaspace）实现，即本地内存Native Memory。
&emsp;&emsp;至于为什么使用本地内存替换永久代，官网有如此描述：This is part of the JRockit and Hotspot convergence effort. JRockit customers do not need to configure the permanent generation (since JRockit does not have a permanent generation) and are accustomed to not configuring the permanent generation.
&emsp;&emsp;具体移除细节，官网由如此描述：The proposed implementation will allocate class meta-data in native memory and move interned Strings and class statics to the Java heap.
&emsp;&emsp;详细信息参见官网地址http://openjdk.java.net/jeps/122

----------------------------------

## java对象
 
&emsp;&emsp;前面聊完java对象身存的载体的具体结构，下面我们聊一聊java对象本身的一些事，需要说明的的是，此处是基于HotSpot探讨java对象在堆空间上实例的创建、布局以及访问。
1、对象的创建
&emsp;&emsp;从语言层面来讲，java对象的创建就是一个new关键字，当应用运行时，虚拟机遇到一条new指令，将进行如下操作：
&emsp;&emsp;##检查类的加载：检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经被加载、解析和初始化。
&emsp;&emsp;##分配内存：java堆内存是否规整决定了如何分配内存，规整的内存，采用“指针碰撞”（Bump the Pointer），即从已用内存和空闲内存的指针分界点开始向空闲内存移动与对象大小相等的距离即可；如何内存不规整，则需要维护空闲列表（Free List），分配时从列表中选择足够大的空间划分给对象的实例。java堆是否规整与垃圾采集器是否带有压缩整理功能有关，如使用Serial、ParNew这种带Compact过程的收集器时采用指针碰撞，使用CMS这种基于Mark-Sweep算法的收集器时采用空闲列表。另外一个需要考虑的问题是线程安全，解决这个问题有两种方案，1是采用CAS保证原子性；2是为每个线程在java堆中预先分配一小块，即本地线程分配缓冲（TLAB），只需在分配新的TLAB时进行同步锁定，-XX：+/-UseTLAB。
&emsp;&emsp;##初始化为0值：此处不包括对象头，这个操作保证了对象的实例可以不赋初始值即可使用。
&emsp;&emsp;##对象头信息设置：对象头中包含如，这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄、锁等信息。
&emsp;&emsp;##初始化（执行<init>方法）：一般来说，执行new指令之后会接着执行<init>方法（由字节码中是否跟随invokespecial指令决定），从而，一个真正可以使用的对象诞生。
2、对象的内存布局
&emsp;&emsp;HotSpot中，对象在内存中存储的布局分为3部分：对象头（Header）、实例数据（Instance Data）以及对齐填充（Padding）。
&emsp;&emsp;##对象头：对象头包含两部分，1是用于存储对象自身的运行时数据Mark Work，在32位和64位的虚拟机中分别占32bit、64bit,包含信息如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向锁线程ID、偏向时间戳等。eg：32位未锁定状态的对象，25bit用于存储对象哈希码、4bit用于存储对象分代年龄、2bit用于存储锁标识位，1bit固定为0；2是类型指针，即对象指向它的类元数据的指针；对于数组来说，对象头还需有一块用于记录数组长度的数据从而确定数组的大小。
&emsp;&emsp;##实例数据：HotSpot默认的分配策略为longs/doubles、ints、shorts/chars、bytes/boolean、oops（Oddinary Pointers），相同宽度的字段被分配在一起，父类定义的变量位于子类变量之前（如果CompactFields=true，子类较窄的变量也可能插入到父类变量的空隙中）。
&emsp;&emsp;##对齐填充：HotSpot要求对象起始地址必须是8字节的整数倍，对象头正好符合，因此对象实例数据可用对齐填充来补全。
3、对象的访问定位
&emsp;&emsp;java程序需要通过栈上的reference数据来操作堆上的具体对象。
&emsp;&emsp;##使用句柄访问：reference中存储的是对象的句柄地址，句柄中包含了对象实例数据与类型数据各自的具体地址，此方式，java堆划分出特定的内存作为句柄池。对象的定位需要两次指针定位，1次找到句柄、另1次找到具体的对象，好处则在于垃圾回收后，对象并移动，不用更改reference中的地址，只需更改句柄池中对应的地址。
&emsp;&emsp;##使用直接指针访问：reference中存储的是对象地址。对象的定位一次就能找到，提升了性能，HotSpot采用此方案。
  
----------------------------------

## 垃圾回收
 
回收策略
hotspot垃圾回执器

----------------------------------

## 内存调优
 
高性能的垃圾回收