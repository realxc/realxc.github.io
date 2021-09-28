title: springboot-spring-boot-starter-actuator
author: Xiang Chuang
tags:
  - spring
  - springboot
  - 应用监控检查
categories:
  - 爱学爱问
date: 2018-07-12 16:03:00
---
应用监控管理
自定义健康检测器，实现org.springframework.boot.actuate.health.AbstractHealthIndicator
充分利用java.lang.management.ManagementFactory，获取系统资源情况

易极付：
基于spring-boot的应用健康监控：
dubboHealthIndicator      dubbo健康检查
deadlockedThreadsHealthIndicator  死锁线程检查
threadsCountHealthIndicator 线程数量检查，阈值2000
systemLoadHealthIndicator cpu负载检查，阈值1/3
rabbitHealthIndicator rabbitmq可用性检查
sessionYedisHealthIndicator  sessionRedis可用性检查
tomcatHealthIndicator 线程占用数量 阈值1半，   tomcat默认最大线程400
yedisHealthIndicator redis可用性检查
diskSpaceHealthIndicator 硬盘空间检查 阈值management.health.diskspace.threshold=524288000
dataSourceHealthIndicator  使用select 1检查连接 （spring-boot提供）

![upload successful](\images\pasted-17.png)