title: maven-默认jdk版本
author: Xiang Chuang
tags:
  - maven
  - jdk
categories:
  - 爱学爱问
date: 2017-07-20 12:30:00
---
修改maven默认的jdk版本，想改彻底需要在maven的全局配文件（settings.xml）增加以下信息：

在profiles 节点下增加：

![upload successful](\images\pasted-55.png)
<profile>
    <id>jdk-1.6</id>
    <activation>
        <activeByDefault>true</activeByDefault>
        <jdk>1.6</jdk>
    </activation>
    <properties>
        <maven.compiler.source>1.6</maven.compiler.source>
        <maven.compiler.target>1.6</maven.compiler.target>
        <maven.compiler.compilerVersion>1.6</maven.compiler.compilerVersion>
    </properties>
</profile> 