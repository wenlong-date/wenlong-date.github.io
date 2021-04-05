---
layout: post
title: 'CentOS7 MySQL部署安装'
tags: [mysql]
---

# 一、安装
这里使用的是rpm安装，rpm可以从[官网yum文件下载](https://dev.mysql.com/downloads/repo/yum/)

参考安装命令

```
wget https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
rpm -ivh mysql57-community-release-el7-9.noarch.rpm
yum -y  install mysql-server 

```

# 二、修改密码及设置远程登录
```
查看临时密码
grep 'temporary password' /var/log/mysqld.log

进入MySQL
mysql -uroot -p 

修改密码 大小写和特殊字符 数字  大于8位
ALTER USER 'root'@'localhost' IDENTIFIED BY 'youNewpass1$';

添加远程访问用户及权限
GRANT ALL PRIVILEGES ON *.* TO 'remoteuser'@'%' IDENTIFIED BY 'RemotePa1$' WITH GRANT OPTION;
```

# 三、启动及重启
```
启动
systemctl start mysqld

查看状态
systemctl status mysqld

设置开机启动
systemctl enable mysqld

重启
systemctl restart mysqld
```


# 四、常用配置修改

## 4.1 建议修改配置
默认配置文件所在路径：  `/etc/my.cnf`

### 4.1.1 设置字符编码
```
[mysqld]
character-set-server=utf8mb4
[client]
default-character-set=utf8mb4
[mysql]
default-character-set=utf8mb4
```

## 4.2 可选修改配置文件设置

### 4.2.1 设置时区
```
查看默认设置 show VARIABLES like "%time_zone%"  
如若需要在[mysqld]下设置：default_time_zone='+8:00'

```

### 4.2.2 常见日志及配置文件设置

```
错误日志：
如若需要在[mysqld]下设置 
log_error=/path/to/mysql-error.log

SQL执行日志：
如若需要在[mysqld]下设置 
general_log=on
general_log_file=/path/to/mysql-general.log


SQL慢日志
如若需要在[mysqld]下设置 
slow_query_log=on
slow_query_log_file=/path/to/mysql-slow.log
slow_launch_time=1   (second)

binlog日志
如若需要在[mysqld]下设置 
server_id=1
log-bin=mysql-bin

```


# 参考文档：  
[1. MySQL的日志文件及配置](https://zhuanlan.zhihu.com/p/352081453)   
[2. 腾讯云centos7下mysql安装](https://juejin.im/post/5befb028f265da61137edd2b  ) 
