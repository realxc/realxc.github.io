title: grafana安装及使用
author: Xiang Chuang
tags:
  - grafana
categories: []
date: 2019-11-25 15:57:00
---
1、官网下载地址：https://grafana.com/grafana/download
 
 centos：wget https://dl.grafana.com/oss/release/grafana-6.4.4-1.x86_64.rpm 
sudo yum localinstall grafana-6.4.4-1.x86_64.rpm 

mac：brew update 
brew install grafana

2、新增用户
	grafana注册用户需要采用邮件服务器进行激活，如果没有邮件服务器，可更改数据库，直接新增用户，步骤如下：
    1、 ps -ef | grep grafana 找到数据库地址
	2、cfg:default.paths.data=数据库地址
	3、cd 数据库地址
	4、sqlite3 grafana.db
	5、select * from user; 
    6、sqlite> select * from  sqlite_master where type="table" and name = "user"
   ...> ;
table|user|user|6|CREATE TABLE `user` (
`id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL
, `version` INTEGER NOT NULL
, `login` TEXT NOT NULL
, `email` TEXT NOT NULL
, `name` TEXT NULL
, `password` TEXT NULL
, `salt` TEXT NULL
, `rands` TEXT NULL
, `company` TEXT NULL
, `org_id` INTEGER NOT NULL
, `is_admin` INTEGER NOT NULL
, `email_verified` INTEGER NULL
, `theme` TEXT NULL
, `created` DATETIME NOT NULL
, `updated` DATETIME NOT NULL
    7、复制admin用户的相关信息（login，email等），将权限is_admin改为0
    eg： insert into user values(2,0,'tech','technology@yiji.com','tech','04b19245b07f5fe7bd4f4b0dd3567d97b368470e03f7d42079d40c168288a5c9f976191216ae70b2e2aca6192112d98f1fa9','AGlaA8e8UK',null,null,1,0,0,null,'2019-11-06 07:12:14','2019-11-06 07:12:14',0,null,0);
    8、访问ip:3000，使用7设置的用户名和管理员admin的密码登录，此时弹出对话框要求重置密码，进行新账户密码重置，此时该账户没有dashbord查询权限
    9、使用admin管理员登录，
    invite your team
![upload successful](\images\pasted-95.png)
invite

![upload successful](\images\pasted-96.png)

![upload successful](\images\pasted-97.png)

![upload successful](\images\pasted-98.png)
10、重新使用新账户登录即可看到dashbord