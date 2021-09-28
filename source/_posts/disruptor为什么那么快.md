title: disruptor为什么那么快
author: Xiang Chuang
tags:
  - disruptor
categories:
  - 爱学爱问
date: 2018-11-13 17:05:00
---
disruptor可用于线程间数据交互
详细参考文章http://ifeve.com/disruptor/

概述一下为什么那么快（这将有利于你借鉴思想）
1、私有序列号，它是AtomicLong类型的，基于CAS（cpu级指令），因此没有锁
2、缓存行填充，避免伪共享（一个缓存行64字节，对应8个long）
   伪共享：多个线程修改相互独立的变量时，如果它们位于同一个缓存行，则         会影响性能
    java8：@Contended（自动填充缓存行），
        但需要jvm参数-XX:RestrictContended
   注意，缓存行的填充是以存储换性能。。。
3、内存屏障 volatile   
   
