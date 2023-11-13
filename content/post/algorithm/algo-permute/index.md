---
title: "全排列"
description: 
date: 2023-11-13T17:30:56+08:00
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
---
### 题目定义：
给定一个不含重复数字的数组 nums ，返回其 所有可能的全排列 。你可以 按任意顺序 返回答案。

示例 1：

输入：nums = [1,2,3]
输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
示例 2：

输入：nums = [0,1]
输出：[[0,1],[1,0]]
示例 3：

输入：nums = [1]
输出：[[1]]



### 整体算法思路：

采用回溯算法，遍历循环数组，形成路径，每一层中循环的都是数组的所有元素，为了不使元素重复，加入额外的存储空间存储判断。

![image-20231113175057163](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202311131750211.png)


### 题解：
``` java
    public List<List<Integer>> permute(int[] nums) {
        //存储最终结果
        List<List<Integer>> result = new ArrayList<>();
        //临时存储数组中元素是否已被当前路径使用
        boolean[] used = new boolean[nums.length];
        //路径
        LinkedList<Integer> path = new LinkedList<>();
        backTrace(path,used,nums,result);
        return result;
    }
    public void backTrace(LinkedList<Integer> path,boolean[] used,int[] nums,List<List<Integer>> result){
        //当这条路径元素数和数组大小一致时，说明该条路径已走完，存储结果并返回
        if (path.size()==nums.length) {
            result.add(new ArrayList<>(path));
            return;
        }
        for(int i = 0;i< nums.length;i++){
            //当数组元素标记被使用，跳过
            if(used[i]) continue;
            //将元素加入路径中
            path.add(nums[i]);
            //标记元素被使用
            used[i] = true;
            //回溯往下选择路径
            backTrace(path,used,nums,result);
            //往回撤销选择
            used[i] =false;
            //去除路径中已选择的元素
            path.removeLast();
        }
    }
```