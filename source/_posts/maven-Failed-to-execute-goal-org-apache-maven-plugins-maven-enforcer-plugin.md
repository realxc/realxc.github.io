title: 'maven-Failed to execute goal org.apache.maven.plugins:maven-enforcer-plugin:'
author: Xiang Chuang
tags:
  - maven
categories:
  - 爱学爱问
date: 2017-07-20 12:30:00
---
maven带的强制或者自定义得强制，导致打包不过。

info：

util:jar:2.2.20150520
Found Banned Dependency: org.slf4j:slf4j-log4j12:jar:1.7.10
Use 'mvn dependency:tree' to locate the source of the banned dependencies.


禁止依赖org.slf4j:slf4j-log4j12:jar，因此打包不过，需要排除依赖