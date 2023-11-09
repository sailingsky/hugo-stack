---
title: "斐波那契数列解法"
description: 
date: 2023-09-25T10:52:14+08:00
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
    - Fibonacci
---

### 斐波那契数列定义：

> 引用自维基百科：

**斐波那契数**（[意大利语](https://zh.wikipedia.org/wiki/意大利语)：Successione di Fibonacci），又译为**菲波拿契数**、**菲波那西数**、**斐氏数**、**黄金分割数**。所形成的[数列](https://zh.wikipedia.org/wiki/数列)称为**斐波那契数列**（[意大利语](https://zh.wikipedia.org/wiki/意大利语)：Successione di Fibonacci），又译为**菲波拿契数列**、**菲波那西数列**、**斐氏数列**、**黄金分割数列**。这个数列是由[意大利](https://zh.wikipedia.org/wiki/意大利)[数学家](https://zh.wikipedia.org/wiki/數學家)[斐波那契](https://zh.wikipedia.org/wiki/斐波那契)在他的《算盘书》中提出。

在[数学](https://zh.wikipedia.org/wiki/數學)上，**斐波那契数**是以[递归](https://zh.wikipedia.org/wiki/递归)的方法来定义：

- ![F_{0}=0](https://wikimedia.org/api/rest_v1/media/math/render/svg/58ebe8b2d5551fb272cd4258940fe1e492592d02)
- ![F_{1}=1](https://wikimedia.org/api/rest_v1/media/math/render/svg/c374ba08c140de90c6cbb4c9b9fcd26e3f99ef56)
- ![F_{n}=F_{{n-1}}+F_{{n-2}}](https://wikimedia.org/api/rest_v1/media/math/render/svg/4fa6d281e7a54e08aeffeef7458ddc0884333686)（![{\displaystyle n\geqq 2}](https://wikimedia.org/api/rest_v1/media/math/render/svg/12e27a3b350c6e2b14091d449563b273d76070a9))

用文字来说，就是斐波那契数列由0和1开始，之后的斐波那契数就是由之前的两数相加而得出。



### 递归解法1：

根据其数学定义，采用递归方式，直译成代码：

```java
 static int fib1(int n){
        if(n==0||n==1) return n;
        return fib1(n-1)+fib1(n-2);
    }
```



此种方式简单明了，但运行起来会发现，随着n的增大，其耗时也随着增大变慢。其中原因在于会重复计算，如n=40时，第一次递归时计算f(39)和f(38)，第二次递归时计算f(38),f(37)以及f(37)和f(36)。为了加快效率，我们可以加入个临时存储空间，将计算过的结果存放与此。

### 递归临时存储：

运用数组存放已计算过的结果，数组长度为n+1;

``` java
    static int fibWithArrayTemp(int n,int[] temp){
        if(n==0||n==1) return n;
        //如果该位置非0，说明已经存在计算结果
        if(temp[n]!=0) return temp[n];
        temp[n] = fibWithArrayTemp(n-1,temp)+fibWithArrayTemp(n-2,temp);
        return temp[n];
    }
```



### 动态规划解法：

以上解法都是利用递归，从上往下进行，采用动态规划，由下往上进行计算。辅助数组`dptable`用于存储计算过程中产生的结果。

``` java
    static int fibWithDP(int n){
        if(n==0) return 0;
        int[] dptable = new int[n+1];
        dptable[0]=0;dptable[1]=1;
        for(int i=2;i<=n;i++){
            dptable[i] = dptable[i-1]+dptable[i-2];
        }
        return dptable[n];
    }
```



### 动态规划解法-优化：

上面的都是通过创建n+1个元素数组，作为临时结果存储。空间复杂度为O(N),那还有没有可能优化一下。当然是可以的，我们可以找出规律，发现有用的都是前两个结果元素，可以只用两个变量存储即可，这样能将空间复杂度降为O(1)。

``` java
    static int fibWithDPOpt(int n ){
        if(n==0) return 0;
        //前后两个元素
        int pre= 0,post = 1 ;
        int result = 0;
        for(int i=2;i<=n;i++){
            result = pre+post;
            //滚动更新前后两个元素结果
            pre = post;
            post = result;
        }
        return result;
    }
```



