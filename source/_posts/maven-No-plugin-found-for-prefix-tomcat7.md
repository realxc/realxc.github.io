title: maven-No plugin found for prefix 'tomcat7'
author: Xiang Chuang
tags:
  - maven
categories:
  - 爱学爱问
date: 2017-07-20 12:30:00
---
没有tomcat7插件
解决：在parent中

 <plugin>
         <groupId>org.apache.tomcat.maven</groupId>
         <artifactId>tomcat7-maven-plugin</artifactId>
         <version>2.2</version>
         <configuration>
             <port>8091</port>
             <uriEncoding>UTF-8</uriEncoding>
             <path>/</path>
         </configuration>
       </plugin>