title: class及javaagent装载简述
author: Xiang Chuang
tags:
  - 类加载
  - javaagent
categories:
  - 爱学爱问
date: 2018-11-01 17:18:00
---
场景：有些时候我们想做一个通用的事情，这关系到各个应用系统，如采集应用系统产生的数据，当然，我们不想代码倾入到应用系统中，“偷偷摸摸、神不知鬼不觉”地干这些事，那么，你可以考虑考虑借鉴javaagent的思想在Class类加载期间干点事。

![upload successful](\images\pasted-79.png)

javaagent装载阶段预置了transformer
class装载阶段读取transformer信息并进行制定的处理

因此，你可以在javaagent中自定义transformer以做你想做的事，如使用bytebuddy代理类（这个代理性能不错，学习成本低，当然如果你熟悉字节码，可以直接asm）
