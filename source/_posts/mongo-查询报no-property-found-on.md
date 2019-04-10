title: mongo-查询报no property found on
author: Xiang Chuang
tags:
  - mongo
  - db
categories:
  - 爱学爱问
date: 2018-12-07 15:11:20
---
No property null found on entity com.yiji.dtrace.core.SpanId to bind constructor parameter to!

现象：线上和本地都没问题，同样的应用同样的数据，就测试环境有问题


如果有自定义转换器走这，没得 就会默认

![upload successful](\images\pasted-6.png)

![upload successful](\images\pasted-7.png)
加载构造函数，这里是通过java8新特性，直接获取到了构造函数的入参名称，然后以此为key获取mongo集合对应的值
![upload successful](\images\pasted-8.png)

![upload successful](\images\pasted-9.png)

![upload successful](\images\pasted-10.png)

![upload successful](\images\pasted-11.png)
根据构造函数的入参获取mongo查出来的值，问题出在constructor.getParameters()获取入参时候， 取得了一个null

![upload successful](\images\pasted-12.png)

No property null found on entity com.yiji.dtrace.core.SpanId to bind constructor parameter to!

问题在于：
测试环境apm启动时，使用了javaagent代理，代理本身就是apm的，里面也有com.yiji.dtrace.core.SpanId，因此出现了冲突！！！！！

java8通过以下类提供了反射获取方法参数名成的能力

![upload successful](\images\pasted-14.png)

![upload successful](\images\pasted-15.png)