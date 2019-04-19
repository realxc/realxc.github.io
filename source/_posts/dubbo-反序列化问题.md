title: dubbo-反序列化问题
author: Xiang Chuang
tags:
  - dubbo
  - 序列化
  - hessian
categories:
  - 这些年，那些坑
date: 2018-06-28 12:14:00
---
问题：网银充值，支付引擎调用清算调用网关，网关响应清算，清算响应支付引擎（支付引擎解析时出现反序列化异常）

原因分析：清算配置了yiji.dubbo.provider.serialization=hessian3，即传输时使用hessian3序列化、反序列化
，但是支付引擎使用的hessian2.  使用hessian2反序列化hessian3序列化出了问题

但是清算反序列化网关为什么没有问题，  3反序列化2做了兼容？还是说，本身就是用的2反序列化2.  如果时2反序列化2，那么即使支付引擎配置成3，那岂不又成了2反序列化3？   所以应该是兼容吧！！！！