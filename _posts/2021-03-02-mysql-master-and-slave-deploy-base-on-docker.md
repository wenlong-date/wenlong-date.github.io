---
layout: post
title: 'MySQL主从简单部署实践'
tags: [mysql]
---

# 一、目的
在实际的生产开发中，当面对单台数据库服务读取请求压力一直持续比较大的场景下，
我们会考虑数据库进行主从部署来分担单台服务的压力。

下面主要是实践了解 MySQL 如何部署主从运行方式。

# 二、基本部署流程
这里选用基于 Docker 运行的方式部署MySQL主从服务，基于 Docker 我们可以很方便的运行起来MySQL服务

## 2.1 启动MySQL的主从两个服务
```
master服务
docker run -p 3301:3306 --name mysql-master -e MYSQL_ROOT_PASSWORD=updateYourPass! -d mysql:5.7

slave服务
docker run -p 3302:3306 --name mysql-slave -e MYSQL_ROOT_PASSWORD=updateYourPass! -d mysql:5.7
```

## 2.2 主从服务配置

执行`docker ps`，可以看到下面
```
CONTAINER ID   IMAGE       COMMAND                  CREATED      STATUS         PORTS                               NAMES
4e5b0107e1f3   mysql:5.7   "docker-entrypoint.s…"   7 days ago   Up 4 hours     33060/tcp, 0.0.0.0:3302->3306/tcp   mysql-slave
a342dc23a334   mysql:5.7   "docker-entrypoint.s…"   7 days ago   Up 4 hours     33060/tcp, 0.0.0.0:3301->3306/tcp   mysql-master

```
### 2.2.1 配置master节点
```
1. 进入Docker容器
docker exec -it a342dc23a334 /bin/bash

2. 安装vim
sed -i "s@http://deb.debian.org@http://mirrors.aliyun.com@g" /etc/apt/sources.list
sed -i "s@http://security.debian.org@http://mirrors.aliyun.com@g" /etc/apt/sources.list
apt-get update 
apt-get install vim 
vim /etc/mysql/my.cnf

3. 增加配置，时区配置根据实际添加
[mysqld]
general_log=on  // 生产环境请关闭
general_log_file=/tmp/mysql-general.log
default_time_zone='+8:00'
log-bin=mysql-bin
server-id=1

4. 退出&重启master
容器内 service mysql restart
宿主机 docker restart a342dc23a334

5. 增加及授权数据同步所需的同步用户
容器内MySQL命令行执行
CREATE USER 'slave'@'%' IDENTIFIED BY 'slaveP455word!';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';

```

### 2.2.2 配置slave节点
```
1. 进入Docker容器
docker exec -it 4e5b0107e1f3 /bin/bash

2. 安装vim
sed -i "s@http://deb.debian.org@http://mirrors.aliyun.com@g" /etc/apt/sources.list
sed -i "s@http://security.debian.org@http://mirrors.aliyun.com@g" /etc/apt/sources.list
apt-get update 
apt-get install vim 
vim /etc/mysql/my.cnf

3. 增加配置，时区配置根据实际添加
[mysqld]
general_log=on   // 生产环境请关闭
general_log_file=/tmp/mysql-general.log
default_time_zone='+8:00'
log-bin=mysql-bin
server-id=2  
read_only=1

4. 退出&重启master
容器内 service mysql restart
宿主机 docker restart a342dc23a334

```

### 2.2.3 设置主从相关配置

```
1. master 节点容器内 MySQL 命令行执行, 获取 master_log_file 文件名和 master_log_pos。
show master status
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+

2. 获取容器的IP地址，在宿主机执行命令

➜  ~ docker inspect --format='{{.NetworkSettings.IPAddress}}' mysql-master
172.17.0.2
➜  ~ docker inspect --format='{{.NetworkSettings.IPAddress}}' mysql-slave
172.17.0.3

3. slave 节点容器内MySQL命令行执行
change master to master_host='172.17.0.2', master_user='slave', master_password='slaveP455word!', master_port=3306, master_log_file='mysql-bin.000001', master_log_pos=154, master_connect_retry=30;

4. slave 节点容器内MySQL命令行执行
start slave;

```

至此 master-slave 方式的MySQL服务就基本配置好了，当我们在 master 节点上进行数据库创建、表创建、数据插入及更新时，也能将数据同步到 slave 节点上的 MySQL 服务。


# 参考内容
[1. 基于Docker部署主从MySQL](https://juejin.cn/post/6907112721532059662) 