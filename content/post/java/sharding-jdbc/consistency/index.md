---
title: "sharding-jdbc实现一致性哈希分片"
description: 
date: 2024-03-23T21:48:33+08:00
image: post/java/sharding-jdbc/pexels-google-deepmind-17483874.jpg
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
之前的sharding-jdbc文章里读写分离水平分片采用了取余的方式，即按照当前实例数简单粗暴区域分片。而取余来说简单易懂，带来的问题也是显而易见的，就是在进行扩容的时候，会导致很多数据的迁移和查询失效。
一致性哈希算法能够避免以上的问题，其扩容也只会影响一小部分数据，具体的一致性哈希可参考网上很多的博客，这里不再赘述。

#### 一致性哈希算法定义
``` java
package com.example.shardingjdbcdemo.sharding;

import lombok.extern.slf4j.Slf4j;
import org.apache.shardingsphere.sharding.api.sharding.standard.PreciseShardingValue;
import org.apache.shardingsphere.sharding.api.sharding.standard.RangeShardingValue;
import org.apache.shardingsphere.sharding.api.sharding.standard.StandardShardingAlgorithm;

import java.util.Collection;
import java.util.SortedMap;
import java.util.TreeMap;

/**
 * @Author: zhangsj
 * @Date: 2024-03-23 20:00
 */
@Slf4j
public class ConsistencyShardingAlgorithm implements StandardShardingAlgorithm<Long> {

    SortedMap<Integer,String> consistentMap = new TreeMap<>();

    @Override
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> shardingValue) {
        if(consistentMap.isEmpty()){
            //根据实际节点，创建出对应256个虚拟节点
            for(String targetName: availableTargetNames){
                for(int j=0;j<256;j++){
                    String key = targetName+"-"+j;
                    consistentMap.put(getHash(key),targetName);
                }
            }
        }
        int shardingHash = getHash(shardingValue.getValue()+"");
        //得到大于该分片hash值的所有map
        SortedMap<Integer,String> subMap = consistentMap.tailMap(shardingHash);
        if(subMap.isEmpty()){
            Integer firstKey = consistentMap.firstKey();
            return consistentMap.get(firstKey);
        }else{
            return subMap.get(subMap.firstKey());
        }
    }

    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<Long> shardingValue) {
        return availableTargetNames;
    }

    @Override
    public void init() {

    }

    @Override
    public String getType() {
        return "Consistency";
    }

    //使用FNV1_32_HASH算法计算服务器的Hash值
    private static int getHash(String str) {

        final int p = 16777619;
        int hash = (int) 2166136261L;
        for (int i = 0; i < str.length(); i++) {
            hash = (hash ^ str.charAt(i)) * p;
        }
        hash += hash << 13;
        hash ^= hash >> 7;
        hash += hash << 3;
        hash ^= hash >> 17;
        hash += hash << 5;

        // 如果算出来的值为负数则取其绝对值
        if (hash < 0) {
            hash = Math.abs(hash);
        }
        log.info("key:{},hash:{}",str,hash);
        return hash;
    }
}

```

#### 配置中设置
由于一致性哈希一般针对数据库来设置，所以一致性哈希分片配置只放在数据库上。关键词`consistency`
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
                sharding-algorithm-name: consistency
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
          #一致性哈希
          consistency:
            type: CLASS_BASED
            props:
              strategy: standard
              algorithmClassName: com.example.shardingjdbcdemo.sharding.ConsistencyShardingAlgorithm
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

当然，分片算法不止可以用标准算法，也可采用其他的分片算法，具体可参照官方文档 [分片](https://shardingsphere.apache.org/document/5.1.0/cn/features/sharding/concept/sharding/) 和 [分片算法](https://shardingsphere.apache.org/document/5.1.0/cn/user-manual/shardingsphere-jdbc/builtin-algorithm/sharding/#%E5%A4%8D%E5%90%88%E5%88%86%E7%89%87%E7%AE%97%E6%B3%95)