title: dubbo-（反）序列化
author: Xiang Chuang
tags:
  - dubbo
  - 序列化
categories:
  - 爱学爱问
date: 2018-06-12 18:24:00
---
dubbo枚举（反）序列化使用的是：com.alibaba.com.caucho.hessian.io.EnumDeserializer
根据枚举的name进行处理Enum.valueOf(Class, enumName);
使用时，只要保证当前应用有对应name的枚举值就能拿到，没有则抛异常

注：
1、rabbit mq序列化默认使用com.esotericsoftware.kryo.serializers.DefaultSerializers.EnumSerializer
kyro按顺序处理
使用时，必须保证序列化和反序列化时，枚举值的顺序是一致的，否则拿到错误的枚举值，同时可能出现下标越界

2、com.alibaba.fastjson.JSON.parseObject.parseJSON使用com.alibaba.fastjson.parser.deserializer.EnumDeserializer
根据枚举的name进行处理Enum.valueOf(Class, enumName);
使用时，只要保证当前应用有对应name的枚举值就能拿到，没有则抛异常