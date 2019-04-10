title: druid-filter
author: Xiang Chuang
tags:
  - 数据库连接池
  - db
  - druid
categories:
  - 爱学爱问
date: 2018-07-12 14:02:00
---
com.alibaba.druid.filter.Filter
过滤器，对数据库操作做切面代理

org.apache.ibatis.session.SqlSession
org.mybatis.spring.SqlSessionTemplate#sqlSessionProxy
com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl@3fadc0bd(DataSourceUtils.getConnection(this.dataSource);)
com.alibaba.druid.pool.DruidPooledPreparedStatement（createChain().preparedStatement_execute(this);）       
com.alibaba.druid.filter.FilterEventAdapter
com.yiji.common.ds.druid.YijiStatFilter（易极付实现）