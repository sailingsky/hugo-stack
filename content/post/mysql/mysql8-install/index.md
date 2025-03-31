---
title: "Mysql8解压包安装"
description: 
date: 2025-03-31T20:00:24+08:00
image: sunset.jpg
math: 
license: 
hidden: false
comments: true
draft: false
---

最近需要装下mysql8，简单做个记录备份下安装，以便以后。

#### 下载压缩安装包：

先去https://dev.mysql.com/downloads/mysql/ 下载对应版本，我选择的是8.4.4，选择zip

![image-20250331200735321](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202503312007533.png)

#### 解压配置：

解压到目标目录，然后在D:\dev\mysql-8.4.4-winx64 根目录下新建mysql的配置文件my.ini

``` ini
[mysqld]
# 设置3309端口
port=3309
# 设置mysql的安装目录，记得切换成自己的路径
basedir=D:\\dev\\mysql-8.4.4-winx64
# 设置mysql数据库的数据的存放目录
datadir=D:\\dev\\mysql-8.4.4-winx64\\data
# 允许最大连接数
max_connections=200
# 允许连接失败的次数。这是为了防止有人从该主机试图攻击数据库系统
max_connect_errors=10
# 服务端使用的字符集默认为UTF8
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
# 默认使用“mysql_native_password”插件认证
mysql_native_password=ON

[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8

[client]
# 设置mysql客户端连接服务端时默认使用的端口
port=3309
default-character-set=utf8


```

打开命令行，运行以下命令：

``` shell
#安装mysql8服务
mysqld --install mysql8

#初始化(注意此后输出中会生成root的默认密码)
mysqld --initialize --user=root --console

#启动mysql8服务
net start mysql8

#进入mysql
mysql -uroot -p

#修改root密码
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password by 'password';
```

![image-20250331202627755](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202503312026042.png)



结束~