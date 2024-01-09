---
title: "Sharding Jdbc实现读写分离+水平分片"
description: 
date: 2024-01-09T21:14:46+08:00
image: pexels-google-deepmind-17483874.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - sharding-jdbc
tags:
    - java
    - sharding-jdbc
---

日常系统开发中，经常涉及到大数据量的业务，常见的如商城系统，规模上去后，单表会经常上千万甚至上亿。仅仅靠单表肯定是无法支撑系统的规模及性能，因此诞生了分库分表的解决方案。首先，为了保证高可用，会增加主从库备份；再者为了高性能，会利用主从库进行读写分离；为了高容量，会进行分库分表。

### 整体架构图：

![image-20240109214707106](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401092147218.png)

`master0`和`master1`同为写库，用于数据分片，增加数据容量。`slave0`和`slave1`为从库，提供查询能力，分担查询压力，提升系统性能。同时作为备份库提高可靠性。



### mysql高可用集群搭建

#### 主mysql搭建
##### 1.docker创建主mysql服务
``` shell
docker run -d \
-p 3306:3306 \
-v /data/mysql/master0/conf:/etc/mysql/conf.d \
-v /data/mysql/master0/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=root \
--name mysql-master \
mysql:8.0.29
```

##### 2.修改主mysql配置文件
``` shell
vim /data/mysql/master0/conf/my.cnf
```
加入以下配置内容:
``` conf
[mysqld]
# 服务器唯一id，默认值1
server-id=1
# 设置日志格式，默认值ROW
binlog_format=STATEMENT
# 二进制日志名，默认binlog
# log-bin=binlog
# 设置需要复制的数据库，默认复制全部数据库
#binlog-do-db=mytestdb
# 设置不需要复制的数据库
#binlog-ignore-db=mysql
#binlog-ignore-db=infomation_schema
```
重启mysql容器：
``` shell
docker restart mysql-master0
```
`binlog格式说明：`

- binlog_format=STATEMENT：日志记录的是主机数据库的`写指令`，性能高，但是now()之类的函数以及获取系统参数的操作会出现主从数据不同步的问题。
  
- binlog_format=ROW（默认）：日志记录的是主机数据库的`写后的数据`，批量操作时性能较差，解决now()或者 user()或者 @@hostname 等操作在主从机器上不一致的问题。
  
- binlog_format=MIXED：是以上两种level的混合使用，有函数用ROW，没函数用STATEMENT，但是无法识别系统变量
##### 3.进入mysql主服务器，修改登录限制
``` shell
#进入容器：env LANG=C.UTF-8 避免容器中显示中文乱码
docker exec -it mysql-master0 env LANG=C.UTF-8 /bin/bash
#进入容器内的mysql命令行
mysql -uroot -p
#修改默认密码校验方式
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
```

##### 4. 创建数据复制所用的slave用户
``` shell
CREATE USER 'slave'@'%';
-- 设置密码
ALTER USER 'slave'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
-- 授予复制权限
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';
-- 刷新权限
FLUSH PRIVILEGES;
```
##### 5. 查看主mysql状态

``` shell
show master status;
```
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401081716103.png)

#### 从服务器搭建
##### 1.docker创建启动从服务器
``` shell
docker run -d \
-p 3307:3306 \
-v /data/mysql/slave0/conf:/etc/mysql/conf.d \
-v /data/mysql/slave0/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=root \
--name mysql-slave1 \
mysql:8.0.29
```

##### 2. 修改从服务器配置
``` shell
vim /data/mysql/slave0/conf/my.cnf
```

修改配置如下：
``` conf
[mysqld]
# 服务器唯一id，每台服务器的id必须不同，如果配置其他从机，注意修改id
server-id=2
# 中继日志名，默认xxxxxxxxxxxx-relay-bin
#relay-log=relay-bin
```
重启容器：
``` shell
docker restart mysql-slave1
```

##### 3. 修改登录方式
``` shell
#进入容器：
docker exec -it mysql-slave0 env LANG=C.UTF-8 /bin/bash
#进入容器内的mysql命令行
mysql -uroot -p
#修改默认密码校验方式
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
```


##### 4. 配置主从关系

``` sql
CHANGE MASTER TO MASTER_HOST='192.168.65.131', 
MASTER_USER='slave',MASTER_PASSWORD='root', MASTER_PORT=3306,
MASTER_LOG_FILE='binlog.000003',MASTER_LOG_POS=1333; 
```

#### slave启动主从同步
``` shell
START SLAVE;
-- 查看状态（不需要分号）
SHOW SLAVE STATUS\G
```



### SpringBoot配置

pom.xml加入`sharding-jdbc`,`mybatis-plus`,`mysql驱动`等依赖

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>sharding-jdbc-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>sharding-jdbc-demo</name>
    <description>sharding-jdbc-demo</description>
    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>2.3.12.RELEASE</spring-boot.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.3.1</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
            <version>5.1.1</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot.version}</version>
                <configuration>
                    <mainClass>com.example.shardingjdbcdemo.ShardingJdbcDemoApplication</mainClass>
                    <skip>true</skip>
                </configuration>
                <executions>
                    <execution>
                        <id>repackage</id>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

```



application.yaml主要涉及sharding-jdbc的配置：

``` yaml
server:
  port: 8080

spring:
  shardingsphere:
    # 内存模式
    mode:
      type: Memory
    datasource:
      names: master0,master1,slave0,slave1
      master0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.65.131:3306/db_order
        username: root
        password: root
      master1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.65.131:3308/db_order
        username: root
        password: root
      slave0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.65.131:3307/db_order
        username: root
        password: root
      slave1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.jdbc.Driver
        jdbc-url: jdbc:mysql://192.168.65.131:3309/db_order
        username: root
        password: root
    #sql日志输出
    props:
      sql-show: true
    rules:
      # 读写分离配置
      readwrite-splitting:
        data-sources:
          rwds0:
            type: Static
            props:
              #写数据源名称
              write-data-source-name: master0
              #读数据源名称
              read-data-source-names: slave0
            # 负载均衡算法名称
            load-balancer-name: alg_round
          rwds1:
            type: Static
            props:
              #写数据源名称
              write-data-source-name: master1
              #读数据源名称
              read-data-source-names: slave1
            # 负载均衡算法名称
            load-balancer-name: alg_round
        # 负载均衡算法类型
        load-balancers:
          alg_round:
            type: RANDOM
      # 分片配置
      sharding:
        tables:
          t_order:
            actual-data-nodes: rwds$->{0..1}.t_order$->{0..1}
            # 分库策略
            database-strategy:
              standard:
                # 分片列名称
                sharding-column: user_id
                #分片算法名称
                sharding-algorithm-name: alg_mod
            # 分表策略
            table-strategy:
              standard:
                sharding-column: id
                sharding-algorithm-name: alg_mod
            #分布式序列策略配置
            key-generate-strategy:
              column: id
              key-generator-name: alg_snowflake
        sharding-algorithms:
          # 分片算法取余，按分片数2取余
          alg_mod:
            type: mod
            props:
              sharding-count: 2
        #主键生成策略
        key-generators:
          alg_snowflake:
            type: SNOWFLAKE
```

测试代码：

``` java
package com.example.shardingjdbcdemo;

import com.example.shardingjdbcdemo.entity.Order;
import com.example.shardingjdbcdemo.mapper.OrderMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.math.BigDecimal;
import java.util.List;

/**
 * @Author: 
 * @Date: 2024-01-09 17:55
 */
@SpringBootTest
public class ShardingTest {
    @Autowired
    OrderMapper orderMapper;
    @Test
    public void testInsertOrder(){
        for(int i =0;i<5;i++){
        Order order = new Order();
        order.setOrderNo("sn20231011");
        order.setUserId(Long.valueOf(i+1));
        order.setAmount(BigDecimal.valueOf(2.22));
        orderMapper.insert(order);
        }
    }

    @Test
    public void query(){
        List<Order> orders = orderMapper.selectList(null);
        List<Order> orders1 = orderMapper.selectList(null);
        List<Order> orders2 = orderMapper.selectList(null);
        List<Order> orders3 = orderMapper.selectList(null);

    }
}

```



可从插入数据查看到对应主从库及分片数据的存储，查询也可根据控制台打印的sql，看到数据的查询走的从库了，分担了主库的压力。

![image-20240109215925715](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202401092159789.png)



当然，你还可以根据上面的架构，改造成一主多从，实现水平分库分表，多从库读的能力。