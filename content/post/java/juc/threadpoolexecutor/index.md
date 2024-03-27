---
title: "Threadpoolexecutor使用不当的bug"
description: 
date: 2024-03-27T11:26:32+08:00
image: neom-LAj90eAXOZA-unsplash.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - JUC
tags:
    - Java
    - JUC
---

ThreadPoolExecutor这个应该很多人都用过，主要用来多线程任务并行执行。当然如果用不好，也会有很多意想不到的坑，比如常见的队列用错导致堆溢出。今天说的不是这个问题，这是之前线上遇到的问题。

#### 代码

线程业务任务：

``` java
public class ThreadTask implements Runnable{
    @Override
    public void run() {
        try {
            //模拟业务代码执行时间1s
            Thread.sleep(1000);
            logger.info(Thread.currentThread().getName()+"  "+System.currentTimeMillis());
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```



业务代码中使用ThreadPoolExecutor线程池：

``` java
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5,10,3L,
                TimeUnit.SECONDS,new ArrayBlockingQueue<>(10));

            for (int i = 0; i < 30; i++) {
                ThreadTask threadTask = new ThreadTask();
                threadPoolExecutor.submit(threadTask);
            }

        threadPoolExecutor.shutdown();
```



这个代码咋看起来，也没有什么问题吧。线程池大小，阻塞队列也用的有界队列，线程池最后也进行了关闭处理。然而，实际情况却是线程池不会关闭，因为代码根本不会执行到关闭那行。这样当代码不断执行，创建的线程池越来越多，线程池却没被关闭，最终拖垮系统。



#### 原因

我们一步步来梳理下：

- 来了30个线程任务，扔到了线程池里
- 核心线程数5，那么先创建5个线程执行任务，还剩25个任务
- 接着扔10个任务到阻塞队列中，还剩15个任务
- 发现还是不够，最大线程数还没超过10，那行，再创建5个线程跑任务，还剩10个任务
- 那这剩下的10个任务就根据拒绝策略处理了。

submit 提交任务后，会调用 execute方法，当发现无法再添加worker执行任务时，就会调用拒绝策略的reject方法：

![image-20240327153700173](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202403271537262.png)

![image-20240327153809277](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202403271538307.png)

问题呢，就出在最后一步的拒绝策略，因为源代码中线程池初始化时，并没有指定具体的拒绝策略，就会采用默认的`defaultHandler`.`defaultHandler`是`AbortPolicy`.

![image-20240327153026955](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202403271530002.png)

看下`AbortPolicy` 里面处理拒绝逻辑是，抛出了个`RejectExecutionException`.

![image-20240327153410412](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202403271534453.png)



很明显，抛出异常后，后面的代码执行就被中断了，`threadPoolExecutor.shutdown()` 这个也就不会被执行了。



#### 优化

1. 将代码用try-catch包起来，像这样：

   ``` java
           try {
               for (int i = 0; i < 30; i++) {
                   ThreadTask threadTask = new ThreadTask();
                   threadPoolExecutor.submit(threadTask);
               }
           }catch (Exception ex){
               log.error(ex.getMessage());
           }finally{
           	threadPoolExecutor.shutdown();
           }
   ```

   

2. 初始化线程池时，指定其他的拒绝策略

   ``` java
           ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(5,10,3L,
                   TimeUnit.SECONDS,new ArrayBlockingQueue<>(10), new ThreadPoolExecutor.DiscardPolicy());
   ```

3. 或者自定义拒绝策略

   ``` java
   public class TaskRejectHandler implements RejectedExecutionHandler {
       @Override
       public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
           logger.error("{} had been reject ",r.toString());
       }
   }
   ```

   

4.加大队列长度，核心线程数大小和最大线程数大小。
