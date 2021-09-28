title: jdk8-dump异常
author: Xiang Chuang
tags:
  - java
  - jdk8
  - dump
categories:
  - 这些年，那些坑
date: 2018-09-10 10:36:00
---
![upload successful](\images\pasted-1.png)
易极付代扣应用疑似存在内存泄漏，因此，尝试进行dump分析，后发现dump一直失败，报错如上，dump命令为：
 jmap -F -dump:format=b,file=盘符:/XXX.hprof <pid>

原因分析：
https://bugs.java.com/view_bug.do?bug_id=8044416



解决方案： 
1、jmap -dump:format=b,file=盘符:/XXX.hprof <pid>，同时应用启动时，jvm参数-XX:+StartAttachListener
    use flag -XX:+StartAttachListener in target java process and make sure you run jmap within the same user as main java process

2、升级jdk版本到1.8.0_60以上8u60(Fixed)9b64(Fixed) 