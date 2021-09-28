title: rocketmq异常分析
author: Xiang Chuang
date: 2020-07-21 20:32:13
tags:
---
1、背景
公司系统中，业务系统与中台系统基于rocketmq进行异步事件交互，基本信息如下：
分组信息，公司rocketmq组件直接使用系统名及环境变量作为groupName
topic信息，目前存在两个topic，XXX及YYY
tag信息，每个topic存在多个tag
2、问题现象
在开发环境测试时（单机），一切运行正常，能收到mq消息
将分支发布到测试服务器后，出现了部分消息收不到
通过console查询到部分消息trackType如下
![upload successful](/images/pasted-105.png)
说明分组A下确实存在未消费的消息
3、问题排查
通过查询rocket日志发现，默认日志路径为${user.home}/logs/rocketmqlogs/rocketmq_client.log

![upload successful](/images/pasted-106.png)
发现该错误，疑似说明消费者没有消费指定的topic

后继续查询console发现同一分组存在两个两个消费者（应用系统两个分支、相同环境变量分别发布在两个服务器），由此分析到，另一个分支中并未消费相应的topic
4、结论
rocketmq按照分组划分消费者，同一分组会根据负载均衡策略消费不同broker分片的消息，如果同一分组下的消费者订阅的topic不一致，将导致部分分片下的消息无法被消费。问题规避，保证同一分组下消费者订阅的topic一致，理论上在测试环境多分支并行开发时易出现该问题