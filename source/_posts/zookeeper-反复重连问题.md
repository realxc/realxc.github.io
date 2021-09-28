title: zookeeper-反复重连问题
author: Xiang Chuang
tags:
  - zookeeper
categories:
  - 这些年，那些坑
date: 2018-09-10 18:09:00
---
现象，zk后台日志发现频繁刷如下日志
Refusing session request for client /192.168.46.4:48551 as it has seen zxid 0x3cdf3127 our last zxid is 0x5bba6c client must try another server

大量刷日志导致zk性能下降，最终导致依赖于zk的服务不可用

经分析：
日志来源代码为 org.apache.zookeeper.server.ZooKeeperServer

![upload successful](\images\pasted-18.png)
lastProcessedZxid在zk中用于记录版本信息，即客户端与服务端版本控制

后经反馈得知，18.09.07由于snet环境zk日志量大，曾情况过zk数据目录数据，当时导致各应用zk不可用，要求各应用重启，后续大部分应用重启完成后业务正常使用，但并没有核对所有系统是否都重启

因此，部分没有重启的应用本地的版本信息还是老的信息，拿此信息与zk服务端交互，由于版本信息不对，导致zk拒绝连接，从而客户端又频繁发起重连尝试。。。

解决，一旦发起过zk数据目录删除，或者迁移，请务必确保所有应用全部重启以获取最新的版本信息