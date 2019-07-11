title: mysql数据库wait_timeout问题
author: Xiang Chuang
tags:
  - mysql
  - 连接池
categories:
  - 这些年，那些坑
date: 2019-07-11 14:59:00
---
背景：近日与猪方合作一项目，项目交付后，对方反馈测试环境老是出现数据库异常（下面给出堆栈日志），并且对方的其他应用没有问题。但交付前，此项目在我们的环境（代码、数据库等都是我们的）中测试并无问题。

问题现象：
```
at java.lang.Thread.run(Thread.java:748)
Caused by: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet successfully received from the server was 56,397 milliseconds ago.  The last packet sent successfully to the server was 0 milliseconds ag
o.
        at sun.reflect.GeneratedConstructorAccessor47.newInstance(Unknown Source)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
        at com.mysql.jdbc.Util.handleNewInstance(Util.java:404)
        at com.mysql.jdbc.SQLError.createCommunicationsException(SQLError.java:981)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3465)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3365)
        at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3805)
        at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2478)
        at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2625)
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2547)
        at com.mysql.jdbc.ConnectionImpl.setAutoCommit(ConnectionImpl.java:4874)
        at com.alibaba.druid.pool.DruidPooledConnection.setAutoCommit(DruidPooledConnection.java:712)
        at org.springframework.jdbc.datasource.DataSourceTransactionManager.doBegin(DataSourceTransactionManager.java:225)
        ... 45 common frames omitted
Caused by: java.io.EOFException: Can not read response from server. Expected to read 4 bytes, read 0 bytes before connection was unexpectedly lost.
        at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2957)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3375)
        ... 53 common frames omitted
2019-07-11 13:41:00.311 [TxId:test_1562754979799^1562754982427^117,SpanId:2404734433174193776] [INFO ] c.z.c.p.b.a.i.w.WithdrawTransImpl:116 - 查询交易结
束TradeQueryResult [sucAmount=0, transDate=null]
2019-07-11 13:42:47.097 [TxId:,SpanId:] [INFO ] c.z.c.c.a.r.DefaultRealtimeMonitor:52 - realtime monitor request end, api server return message : LOOP
2019-07-11 13:42:47.097 [TxId:,SpanId:] [INFO ] c.z.c.c.a.r.DefaultRealtimeMonitor:49 - realtime monitor request begin, url -> http://api.config.zhubajie
.la/realtime/pull
2019-07-11 13:44:00.415 [TxId:192.168.25.11.1015^1561538402946^21456,Span
```
看到这段日志，我注意到“Caused by: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure The last packet successfully received from the server was 56,397 milliseconds ago.  The last packet sent successfully to the server was 0 milliseconds ag
o.”， 56397毫秒，这个数字很扎眼！！！ 因为我们使用的druid连接池，最小生存时间和需要关闭的空闲连接时间都是60s，说明了一个问题，所拿到的连接未超过60s，所以仍在连接池中，但是该连接貌似是失效了。分析到这儿，我猜测猪方mysql数据库的wait_timeout估计是进行了修改且小于56.397秒（默认是8小时，我们工时使用默认值），于是来到猪方的数据库，
```
show global variables like '%timeout';
connect_timeout	10
delayed_insert_timeout	300
have_statement_timeout	YES
innodb_flush_log_at_timeout	1
innodb_lock_wait_timeout	120
innodb_rollback_on_timeout	OFF
interactive_timeout	30
lock_wait_timeout	31536000
net_read_timeout	5
net_write_timeout	5
rpl_semi_sync_master_timeout	10000
rpl_stop_slave_timeout	31536000
slave_net_timeout	60
thread_pool_idle_timeout	60
tokudb_last_lock_timeout	
tokudb_lock_timeout	4000
wait_timeout	30
```
好吧，猜想正确，他们讲wait_timeout设置成了30s。

同时他们自己的连接池配置是
```
 dataSource.setTimeBetweenEvictionRunsMillis(5000);
 dataSource.setMinEvictableIdleTimeMillis(30000);
```
而我们的连接池配置是
```
 dataSource.setTimeBetweenEvictionRunsMillis(60000);
 dataSource.setMinEvictableIdleTimeMillis(60000);
```

问题解决！！！

关于wait_timeout，这个参数到底要不要重新设置值没有固定的标准，保障数据库的可用性和性能即可，但应用层必须根据wait_timeout的实际情况进行相应的配置：
1、最大空闲时间不能大于wait_timeout
2、可以尝试在数据库连接中增加autoReconnect=true
3、testWhileIdle="true" timeBetweenEvictionRunsMillis="10000" validationQuery="select 1"

猪方连接池：
![upload successful](\images\pasted-92.png)
我方连接池：

![upload successful](\images\pasted-94.png)