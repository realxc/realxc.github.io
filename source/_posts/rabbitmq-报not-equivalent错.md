title: rabbitmq-报not equivalent错
author: Xiang Chuang
tags:
  - rabbitmq
  - 消息队列
categories:
  - 这些年，那些坑
date: 2018-07-19 17:28:00
---
问题：
支付引擎往monitor发送监控mq消息时，报错：
2018-07-19 15:54:35.423 ERROR [AMQP Connection 10.21.30.153:5672] CachingConnectionFactory-- Channel shutdown: channel error; protocol method: #method<channel.close>(reply-code=406, reply-text=PRECONDITION_FAILED - parameters for queue 'queue.monitor.logmsg' in vhost '/' not equivalent, class-id=50, method-id=10)
Caused by: com.rabbitmq.client.ShutdownSignalException: channel error; protocol method: #method<channel.close>(reply-code=406, reply-text=PRECONDITION_FAILED - parameters for queue 'queue.monitor.logmsg' in vhost '/' not equivalent, class-id=50, method-id=10)

原因：
支付引擎使用monitor客户端发送mq消息，由如下逻辑：
![upload successful](\images\pasted-43.png)

![upload successful](\images\pasted-44.png)

![upload successful](\images\pasted-45.png)

![upload successful](\images\pasted-46.png)
问题出在：new Queue(XXX);
我们来看看参数：true、false、false。  但是mq服务器上当前的队列是monitor系统通过spring集成创建的，参数为：
false、false、true  同样的队列名，参数不一致，mq会报上述错误
![upload successful](\images\pasted-47.png)

![upload successful](\images\pasted-48.png)
解决方案：
合理的方案：保持参数一致
临时方案：停止monitor，这是队列自动删除了，启动支付引擎，创建队列，再启动monitor，因为monitor是采用spring方式创建队列，不会尝试重新创建