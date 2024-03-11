---
title: "Leetcode-70 爬楼梯"
description: 
date: 2024-03-11T21:39:57+08:00
image: post/algorithm/pexels-google-deepmind-18069816.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - 算法
tags:
    - 算法
    - leetcode
---

#### 题目描述

假设你正在爬楼梯。需要 `n` 阶你才能到达楼顶。

每次你可以爬 `1` 或 `2` 个台阶。你有多少种不同的方法可以爬到楼顶呢？

 

**示例 1：**

```
输入：n = 2
输出：2
解释：有两种方法可以爬到楼顶。
1. 1 阶 + 1 阶
2. 2 阶
```



#### 解题思路

当n=1, 有1种（1阶）方法，即f(1)=1;

当n=2, 有2种（1 阶 + 1 阶;2阶）方法，即f(2)=2;

当n=3, 有3种(1 阶 + 1 阶+1阶;2阶+1阶；1阶+2阶)方法，即f(3)=3=f(2)+f(1);

利用数学归纳法，可得f(n)=f(n-1)+f(n-2),典型的斐波那契数列;



#### 代码

``` java
public int climbStairs(int n) {
        int first= 1,second=1;
        for(int i =1 ;i<n;i++){
            int third = first+second;
            first=second;
            second=third;
        }
        return second;

    }
```



#### 效果

![image-20240311215111026](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202403112151065.png)