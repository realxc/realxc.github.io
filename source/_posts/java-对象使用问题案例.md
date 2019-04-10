title: java-对象使用问题案例
author: Xiang Chuang
tags:
  - java
  - java对象
categories:
  - 爱学爱问
date: 2018-08-06 17:23:00
---
关于java对象的使用，java编码的一个基本能力，但事实上，业务开发中，往往会有疏忽，下面从易极付支付引擎一个bug再次剖析java对象的使用。

业务场景：支付引擎转账-定时任务批量加载挂起的数据执行转账并异步回执
                1、批量查询挂起的转账数据，list
                2、循环遍历list执行转换并回执
                    2.1、将数据转换到业务上下文中
                    2.2、执行转账（数据从业务上下文中获取）
                    2.3、发布转账回执事件（数据从业务上下文中获取）
                 3、在新线程中异步回执（订阅到2.3发布的回执事件后与2并行执行）

上述是简化后的业务流程，下面加以核心代码说明支付引擎的实现逻辑
                1、批量查询挂起的转账数据，list 
              
![upload successful](\images\pasted-24.png)

![upload successful](\images\pasted-25.png)
 2、循环遍历list执行转换并回执
     2.1、将数据转换到业务上下文中
     
![upload successful](\images\pasted-26.png)

![upload successful](\images\pasted-27.png)

![upload successful](\images\pasted-28.png)
这儿调用的是易极付common-util中提供的对象转换方法，源码如下：

![upload successful](\images\pasted-30.png)

![upload successful](\images\pasted-31.png)

 2.2、执行转账（数据从业务上下文中获取）
![upload successful](\images\pasted-32.png)

2.3、发布转账回执事件（数据从业务上下文中获取）
![upload successful](\images\pasted-33.png)

![upload successful](\images\pasted-34.png)

![upload successful](\images\pasted-35.png)

![upload successful](\images\pasted-36.png)

![upload successful](\images\pasted-37.png)

3、在新线程中异步回执（订阅到2.3发布的回执事件后与2并行执行）
![upload successful](\images\pasted-38.png)

上述逻辑，经过实际测试，存在严重问题，问题如下：
异步回执时，拿到的数据并不是实际预期的数据，而被篡改了，可以看见的是，执行异步回执时，拿到了list循环遍历后续的数据。

分析问题，很明显，我们看到了2.1这么一行代码：
![upload successful](\images\pasted-39.png)
那么，当前2.2转账完成后，发出2.3异步回执事件，3执行回执是异步的（并行的），此时下一次循环完全可能已经再次执行了2.1这行代码，那么就有问题！！！  为什么？

上述判断只是初次分析，但经过进一步分析，我们发现其实3中的数据已经在2.3中将数据封装好，3只是从封装好的数据中取出相应的数据而已，2.3是同步执行过程，那么，下一次2.1执行时，一定是在上一次2.3执行完成之后！

但是，经测试，这儿确实是有问题的，
于是，再次分析：
2.1进行数据转换时，通过源码分析，我们得出，此处只是将数据的值一个一个的set到业务上下文的数据对象中，那么意味着什么？  业务上下午serviceContext是一个全局对象，进入执行逻辑前已经初始化完成，那么，其内存已经开辟完成，意味着该对象中数据对象的内存也已开辟完成。那么，set值只是为那一块内存进行了赋值操作
2.3通过getEntityObject将数据赋给新的对象I，那么，此时I和业务上下午中的数据对象都是指向同一块内存区域
3/2.1 此时3和下一个2.1并行执行，3中获取数据对象的值，而1.2给上下文数据对象赋予新的值， 那么，3是从这块内存区域取值，1.2是给这块内存区域赋值，一旦1.2执行在前，那么3拿到的当然是重新赋予的值。

通过上述分析，我们知道了问题根源，那么，如何改造？
很简单
修改2.1的代码为
![upload successful](\images\pasted-40.png)

![upload successful](\images\pasted-41.png)
很显然，同样是给业务上下午数据对象赋值，与上面的方案区别在于，此处是将上下午数据对象指向的新的内存地址。
那么执行2.3时，通过getEntityObject将数据赋给新的对象I，其所指向的内存地址也是这块新的地址
执行3/2.1时，3依旧从这块内存地址取值，而2.1则是将上下午数据对象指向另一块内存地址，毫不冲突，更谈不上数据篡改。

下面图示的方式给与说明：
![upload successful](\images\pasted-42.png)
