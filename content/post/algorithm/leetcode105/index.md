---
title: "Leetcode-105 从前序与中序遍历序列构造二叉树"
description: 
date: 2024-03-11T21:25:08+08:00
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

给定两个整数数组 `preorder` 和 `inorder` ，其中 `preorder` 是二叉树的**先序遍历**， `inorder` 是同一棵树的**中序遍历**，请构造二叉树并返回其根节点。

**示例 1:**

![img](https://assets.leetcode.com/uploads/2021/02/19/tree.jpg)

```
输入: preorder = [3,9,20,15,7], inorder = [9,3,15,20,7]
输出: [3,9,20,null,null,15,7]
```



#### 解题思路

- 利用规则，前序数组规则是：根-左-右 ，中序数组规则是：左-根-右
- 以前序数组为主，确定好左子树的长度（而左子树的长度，可以从中序数组中根左边数字长度获得），接着就能分别对左右子树进行递归构造。

#### 代码

``` java
    /**
     *
     * @param preorder  前序数组
     * @param preStart 前序数组开始坐标
     * @param preEnd   前序数组结束坐标
     * @param inStart  中序数组开始坐标
     * @param inMap
     * @return
     */
    private TreeNode buildTree(int[] preorder,int preStart,int preEnd,int inStart,Map<Integer,Integer> inMap){
        if(preStart>preEnd) return null;
        TreeNode root = new TreeNode(preorder[preStart]);
        //根节点在中序数组中的位置(根据中序规则，根节点左边的都为左子树)
        int rootIdx = inMap.get(preorder[preStart]);
        //中序数组中左子树长度
        int leftLen = rootIdx-inStart;
        //构建左子树
        root.left = buildTree(preorder,preStart+1,preStart+leftLen,inStart,inMap);
        //右子树
        root.right = buildTree(preorder,preStart+leftLen+1,preEnd,rootIdx+1,inMap);
        return root;
    }

	//前序数组规则：根-左-右；中序数组规则：左-根-右
    public TreeNode buildTree(int[] preorder, int[] inorder) {
        //中序数组辅助map，存放数组元素和其对应的位置
        Map<Integer,Integer> inMap =  new HashMap<>();
        for (int i=0;i<inorder.length;i++)
            inMap.put(inorder[i], i);
        return buildTree(preorder,0, preorder.length, 0,inMap);
    }


```

#### 效果：

![image-20240311213609410](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202403112136460.png)
