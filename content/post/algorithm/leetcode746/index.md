---
title: "Leetcode-746 最小代价爬楼梯"
description: 
date: 2024-03-11T21:56:55+08:00
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

给你一个整数数组 `cost` ，其中 `cost[i]` 是从楼梯第 `i` 个台阶向上爬需要支付的费用。一旦你支付此费用，即可选择向上爬一个或者两个台阶。

你可以选择从下标为 `0` 或下标为 `1` 的台阶开始爬楼梯。

请你计算并返回达到楼梯顶部的最低花费。

 

**示例 1：**

```
输入：cost = [10,15,20]
输出：15
解释：你将从下标为 1 的台阶开始。
- 支付 15 ，向上爬两个台阶，到达楼梯顶部。
总花费为 15 。
```

#### 解题思路

和之前`leetcode-70 爬楼梯`类似，第i个阶梯的最小代价公式为:d(i)=min(d(i-1),d(i-2))+c(i)

#### 代码

``` java
public int minCostClimbingStairs(int[] cost) {

        if(cost==null||cost.length==0) return 0;

        if(cost.length==1) return cost[0];
	//前一，二个阶梯的代价
        int first =cost[0],second=cost[1];

        for(int i=2;i<cost.length;i++){
		//当前楼梯的最小代价
            int cur = Math.min(first,second)+cost[i];

            first=second;

            second=cur;

        }

        return Math.min(first,second);

    }
```





#### 效果

![image-20240311220413687](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202403112204724.png)