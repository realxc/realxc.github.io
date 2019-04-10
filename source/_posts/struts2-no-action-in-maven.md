title: struts2-no action in maven
author: Xiang Chuang
tags:
  - struts2
categories:
  - 爱学爱问
date: 2017-07-20 12:30:00
---
原因：convention插件默认不扫描jar中的action
 解决方案：http://struts.apache.org/release/2.3.x/docs/convention-plugin.html#ConventionPlugin-Actionsinjarfiles(struts2convention插件说明)
    <!-- the jar URL can contain a path to the jar file and a trailing "!/" -->
    <constant name="struts.convention.action.includeJars" value=".*?/Imanage\w*.*?jar(!/)?"/>  匹配 前缀为Imanage的所有jar
    
![upload successful](\images\pasted-56.png)