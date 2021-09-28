title: git-本地commit、push、show、reset操作
author: Xiang Chuang
tags:
  - git
  - 版本控制
categories:
  - 爱学爱问
date: 2019-04-28 11:06:00
---
问题：
使用git commit代码时，git commit完成
后续执行git push origin master:master 提示“Everything up-to-date”
使用git show查询，确实能看到提交的内容

原因：
尝试提交到master，但本地对应的其实是一个分支

解决：
切换分支到主干，此时发现，git show，刚刚提交的内容看不到了，且本地代码中也没有了，说明commit关联到了对应的分支

使用git reflog返回如下
fb3e1b1 (HEAD -> master) HEAD@{0}: reset: moving to fb3e1b1
2d7b28e (origin/master, origin/HEAD) HEAD@{1}: checkout: moving from fb3e1b16278856b6108cfe3010f7e4fd8c4e3634 to master
fb3e1b1 (HEAD -> master) HEAD@{2}: commit: 场景化业务监控模型搭建
2d7b28e (origin/master, origin/HEAD) HEAD@{3}: checkout: moving from master to 2d7b28e3^0
2d7b28e (origin/master, origin/HEAD) HEAD@{4}: commit: 引入规则引擎
6fe0e4e HEAD@{5}: commit: 基于elk-kafka数据传输通道活性检测
b28b71b HEAD@{6}: commit: kafka消费者数量及分组在hera中配置，同时采集入口采用线程池执行以提高吞吐量
ad4778d HEAD@{7}: commit: 基于elk-kafka传输通道的数据采集功能实现（需进行性能评估）
3d9affa HEAD@{8}: commit: 基于elk-kafka传输通道的数据采集功能实现（需进行性能评估）
6508afa HEAD@{9}: commit: 后台管理实现类同步到项目中
dd0138f HEAD@{10}: commit: hera配置更改
29764a1 HEAD@{11}: commit: README
75debd5 HEAD@{12}: clone: from http://gitlab.yiji/yangxi/godeye.git

再次使用git reset --hard fb3e1b1，将本地代码还原到了相应的版本，代码终于回来了，吓一跳

$ git reset --hard fb3e1b1
HEAD is now at fb3e1b1 场景化业务监控模型搭建


