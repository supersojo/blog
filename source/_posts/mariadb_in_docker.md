---
title: 如何用docker部署mariadb
date: 2021-9-3 20:13:12
tags:
- docker
- mariadb
categories:
- tools
---

本来的mariadb是部署在host上的，考虑到可迁移性，所以采用docker方案部署mariadb。部署mariadb的过程非常简单，但是中间还是有需到几个问题。这里一一列举记录下来。

## 使用portainer

portainer提供web
ui方便的操作docker。需要把端口publish出来，这样host才可以连接。

## mariadb操作
```
$>mysql -u root -p
#mariadb> create database db1;
#mariadb> create user 'user1'$'localhost' identified by 'pass1';
#mariadb> grant all on db1.* to 'user1'@'%' identified by 'pass1' with grant
option;
#mariadb> flush privileges;
#mariadb> eixt;

```
只要按上面grant权限才能连接，否则不能连接。

## 数据库备份与恢复
```
docker exec containerid /usr/bin/mysqldump -u user1 --password=pass1 db1
>backup.sql

cat backup.sql | docker exec -i containerid /usr/bin/mysql -u user1
--password=pass1 db1
```

## References







