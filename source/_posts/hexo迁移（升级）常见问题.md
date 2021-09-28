title: hexo迁移（升级）常见问题
author: Xiang Chuang
tags:
  - hexo
categories:
  - hexo安装
date: 2021-09-28 11:49:00
---
hexo迁移或升级遇到的问题

hexo的使用问题常见于版本不兼容导致，因此，可从hexo、npm、node版本是否匹配入手分析

1、hexo deploy时，The “mode“ argument must be integer. Received an instance of Object
排查步骤参考：
a、hexo -v 查看hexo相关依赖的版本

![upload successful](/images/pasted-107.png)
根据当前hexo及node版本在hexo官网核对版本是否匹配

hexo的升级
npm uninstall hexo先卸载当前版本
npm install hexo@

node版本更换