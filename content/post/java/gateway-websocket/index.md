---
title: "Spring cloud gateway转发websocket请求"
description: 
date: 2023-11-22T16:45:23+08:00
image: chris-rosiak-YSwcI_rmCYo-unsplash.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - websocket
    - spring cloud gateway
tags:
    - java
    - websocket
    - spring cloud gateway
---
### websocket相关配置：

新建了个websocket-server模块微服务。

- 引入websocket依赖：
``` xml
        <!-- websocket -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
```

- 加入配置代码：

  ``` java
  @Configuration
  public class WebSocketConfig {
  
      @Bean
      public ServerEndpointExporter serverEndpointExporter(){
          return new ServerEndpointExporter();
      }
  }
  
  ```

  

- 加入服务接口逻辑：

  ``` java
  @ServerEndpoint("/ws/alert/{userId}")
  @Component
  @Slf4j
  public class WebSocketServer {
  
      /**
       * concurrent包的线程安全Set，用来存放每个客户端对应的MyWebSocket对象。
       */
      private static ConcurrentHashMap<String, WebSocketServer> webSocketMap = new ConcurrentHashMap<>();
      /**
       * 与某个客户端的连接会话，需要通过它来给客户端发送数据
       */
      private Session session;
      /**
       * 接收userId
       */
      private String userId = "";
  
      /**
       * 连接建立成功调用的方法
       */
      @OnOpen
      public void onOpen(Session session, @PathParam("userId") String userId) {
          this.session = session;
          this.userId = userId;
          if (webSocketMap.containsKey(userId)) {
              webSocketMap.remove(userId);
          }
          webSocketMap.put(userId, this);
          log.info("用户连接:{}" , userId );
      }
  
      /**
       * 连接关闭调用的方法
       */
      @OnClose
      public void onClose() {
          if (webSocketMap.containsKey(userId)) {
              webSocketMap.remove(userId);
          }
          log.info("用户退出:{}" + userId);
      }
      /**
       * 收到客户端消息后调用的方法
       *
       * @param message 客户端发送过来的消息
       */
      @OnMessage
      public void onMessage(String message, Session session) {
          log.info("收到用户消息:{},报文:{}" , userId ,message);
      }
  
      /**
       * @param session
       * @param error
       */
      @OnError
      public void onError(Session session, Throwable error) {
          log.error("用户错误:" + this.userId + ",原因:" + error.getMessage());
          error.printStackTrace();
      }
  
      /**
       * 实现服务器主动推送
       */
      public void sendMessage(String message) {
          webSocketMap.forEach((key, webSocketServer) -> {
              try {
                  webSocketServer.session.getBasicRemote().sendText(message);
              } catch (IOException e) {
                  throw new RuntimeException(e);
              }
          });
      }
  
  }
  ```

  

### gateway配置：

- application.yml加入路由配置：

  ``` yaml
      gateway:
        routes:
          - id: wsroute
  #          uri: ws://websocket-server
            uri: ws://localhost:10119
            predicates:
              - Path=/websocket-server/ws/**
              - Header=Connection,Upgrade
            filters:
              - StripPrefix=1
  ```



### 趟坑排雷：

当你觉得万无一失，启动相关服务进行调试请求的时候，问题就来了：

#### io.undertow.server.HttpServerExchange cannot be cast to reactor.netty.http.server.HttpServerResponse 异常

完整异常堆栈信息如下：

``` log
java.lang.ClassCastException: io.undertow.server.HttpServerExchange cannot be cast to reactor.netty.http.server.HttpServerResponse
	at org.springframework.web.reactive.socket.server.upgrade.ReactorNettyRequestUpgradeStrategy.upgrade(ReactorNettyRequestUpgradeStrategy.java:163)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Error has been observed at the following site(s):
	*__checkpoint ⇢ org.springframework.cloud.gateway.filter.WeightCalculatorWebFilter [DefaultWebFilterChain]
	*__checkpoint ⇢ HTTP GET "/websocket-server/ws/alert/1" [ExceptionHandlingWebHandler]
Original Stack Trace:
		at org.springframework.web.reactive.socket.server.upgrade.ReactorNettyRequestUpgradeStrategy.upgrade(ReactorNettyRequestUpgradeStrategy.java:163)
		at org.springframework.web.reactive.socket.server.support.HandshakeWebSocketService.lambda$handleRequest$1(HandshakeWebSocketService.java:243)
		at reactor.core.publisher.FluxFlatMap.trySubscribeScalarMap(FluxFlatMap.java:152)
		at reactor.core.publisher.MonoFlatMap.subscribeOrReturn(MonoFlatMap.java:53)
		at reactor.core.publisher.InternalMonoOperator.subscribe(InternalMonoOperator.java:57)
		at reactor.core.publisher.MonoDefer.subscribe(MonoDefer.java:52)
		at reactor.core.publisher.MonoDefer.subscribe(MonoDefer.java:52)
		at reactor.core.publisher.MonoIgnoreThen$ThenIgnoreMain.subscribeNext(MonoIgnoreThen.java:236)
		at reactor.core.publisher.MonoIgnoreThen$ThenIgnoreMain.onComplete(MonoIgnoreThen.java:203)
		at reactor.core.publisher.FluxPeek$PeekSubscriber.onComplete(FluxPeek.java:260)
		at reactor.core.publisher.FluxMap$MapSubscriber.onComplete(FluxMap.java:142)
		at reactor.core.publisher.MonoNext$NextSubscriber.onComplete(MonoNext.java:102)
		at reactor.core.publisher.MonoNext$NextSubscriber.onNext(MonoNext.java:83)
		at reactor.core.publisher.FluxDematerialize$DematerializeSubscriber.onNext(FluxDematerialize.java:98)
		at reactor.core.publisher.FluxDematerialize$DematerializeSubscriber.onNext(FluxDematerialize.java:44)
		at reactor.core.publisher.FluxFlattenIterable$FlattenIterableSubscriber.drainAsync(FluxFlattenIterable.java:421)
		at reactor.core.publisher.FluxFlattenIterable$FlattenIterableSubscriber.drain(FluxFlattenIterable.java:686)
		at reactor.core.publisher.FluxFlattenIterable$FlattenIterableSubscriber.onNext(FluxFlattenIterable.java:250)
		at reactor.core.publisher.FluxSwitchIfEmpty$SwitchIfEmptySubscriber.onNext(FluxSwitchIfEmpty.java:74)
		at reactor.core.publisher.Operators$MonoSubscriber.complete(Operators.java:1816)
		at reactor.core.publisher.MonoFlatMap$FlatMapInner.onNext(MonoFlatMap.java:249)
		at reactor.core.publisher.MonoIgnoreThen$ThenIgnoreMain.complete(MonoIgnoreThen.java:284)
		at reactor.core.publisher.MonoIgnoreThen$ThenIgnoreMain.onNext(MonoIgnoreThen.java:187)
		at reactor.core.publisher.MonoIgnoreThen$ThenIgnoreMain.subscribeNext(MonoIgnoreThen.java:232)
		at reactor.core.publisher.MonoIgnoreThen$ThenIgnoreMain.onComplete(MonoIgnoreThen.java:203)
		at reactor.core.publisher.MonoIgnoreElements$IgnoreElementsSubscriber.onComplete(MonoIgnoreElements.java:89)
		at reactor.core.publisher.FluxPeek$PeekSubscriber.onComplete(FluxPeek.java:260)
		at reactor.core.publisher.FluxDematerialize$DematerializeSubscriber.onComplete(FluxDematerialize.java:121)
		at reactor.core.publisher.FluxDematerialize$DematerializeSubscriber.onNext(FluxDematerialize.java:91)
		at reactor.core.publisher.FluxDematerialize$DematerializeSubscriber.onNext(FluxDematerialize.java:44)
		at reactor.core.publisher.FluxIterable$IterableSubscription.fastPath(FluxIterable.java:340)
		at reactor.core.publisher.FluxIterable$IterableSubscription.request(FluxIterable.java:227)
		at reactor.core.publisher.FluxDematerialize$DematerializeSubscriber.request(FluxDematerialize.java:127)
		at reactor.core.publisher.FluxPeek$PeekSubscriber.request(FluxPeek.java:138)
		at reactor.core.publisher.MonoIgnoreElements$IgnoreElementsSubscriber.onSubscribe(MonoIgnoreElements.java:72)
		at reactor.core.publisher.FluxPeek$PeekSubscriber.onSubscribe(FluxPeek.java:171)
		at reactor.core.publisher.FluxDematerialize$DematerializeSubscriber.onSubscribe(FluxDematerialize.java:77)
		at reactor.core.publisher.FluxIterable.subscribe(FluxIterable.java:165)
		at reactor.core.publisher.FluxIterable.subscribe(FluxIterable.java:87)
		at reactor.core.publisher.Mono.subscribe(Mono.java:4400)
		at reactor.core.publisher.MonoIgnoreThen$ThenIgnoreMain.subscribeNext(MonoIgnoreThen.java:255)
		at reactor.core.publisher.MonoIgnoreThen.subscribe(MonoIgnoreThen.java:51)
		at reactor.core.publisher.MonoFlatMap$FlatMapMain.onNext(MonoFlatMap.java:157)
		at reactor.core.publisher.Operators$MonoSubscriber.complete(Operators.java:1816)
		at reactor.core.publisher.MonoCollectList$MonoCollectListSubscriber.onComplete(MonoCollectList.java:128)
		at reactor.core.publisher.DrainUtils.postCompleteDrain(DrainUtils.java:132)
		at reactor.core.publisher.DrainUtils.postComplete(DrainUtils.java:187)
		at reactor.core.publisher.FluxMaterialize$MaterializeSubscriber.onComplete(FluxMaterialize.java:141)
		at reactor.core.publisher.FluxTake$TakeSubscriber.onComplete(FluxTake.java:153)
		at reactor.core.publisher.FluxTake$TakeSubscriber.onNext(FluxTake.java:133)
		at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onNext(FluxOnErrorResume.java:79)
		at reactor.core.publisher.SerializedSubscriber.onNext(SerializedSubscriber.java:99)
		at reactor.core.publisher.FluxTimeout$TimeoutMainSubscriber.onNext(FluxTimeout.java:180)
		at reactor.core.publisher.Operators$MonoSubscriber.complete(Operators.java:1816)
		at reactor.core.publisher.MonoCollectList$MonoCollectListSubscriber.onComplete(MonoCollectList.java:128)
		at org.springframework.cloud.commons.publisher.FluxFirstNonEmptyEmitting$FirstNonEmptyEmittingSubscriber.onComplete(FluxFirstNonEmptyEmitting.java:325)
		at reactor.core.publisher.FluxSubscribeOn$SubscribeOnSubscriber.onComplete(FluxSubscribeOn.java:166)
		at reactor.core.publisher.FluxIterable$IterableSubscription.fastPath(FluxIterable.java:362)
		at reactor.core.publisher.FluxIterable$IterableSubscription.request(FluxIterable.java:227)
		at reactor.core.publisher.FluxSubscribeOn$SubscribeOnSubscriber.requestUpstream(FluxSubscribeOn.java:131)
		at reactor.core.publisher.FluxSubscribeOn$SubscribeOnSubscriber.onSubscribe(FluxSubscribeOn.java:124)
		at reactor.core.publisher.FluxIterable.subscribe(FluxIterable.java:165)
		at reactor.core.publisher.FluxIterable.subscribe(FluxIterable.java:87)
		at reactor.core.publisher.Flux.subscribe(Flux.java:8469)
		at reactor.core.publisher.FluxFlatMap.trySubscribeScalarMap(FluxFlatMap.java:200)
		at reactor.core.publisher.MonoFlatMapMany.subscribeOrReturn(MonoFlatMapMany.java:49)
		at reactor.core.publisher.Flux.subscribe(Flux.java:8455)
		at reactor.core.publisher.FluxFlatMap.trySubscribeScalarMap(FluxFlatMap.java:200)
		at reactor.core.publisher.MonoFlatMapMany.subscribeOrReturn(MonoFlatMapMany.java:49)
		at reactor.core.publisher.FluxFromMonoOperator.subscribe(FluxFromMonoOperator.java:76)
		at reactor.core.publisher.FluxSubscribeOn$SubscribeOnSubscriber.run(FluxSubscribeOn.java:194)
		at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:84)
		at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:37)
		at java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:266)
		at java.util.concurrent.FutureTask.run(FutureTask.java)
		at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180)
		at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293)
		at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
		at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
		at java.lang.Thread.run(Thread.java:750)

```

好了，有异常，咱们网上找找答案，可在网上翻了大半天，发现没几个答案，绝大部分说的都是啥移除tomcat依赖[springcloud gateway+websocket使用的坑](https://blog.csdn.net/weixin_43260335/article/details/123132774)，实际上尝试了并不行。后面翻到了博客园上这哥们的文章[springCloudGateway-踩坑记录](https://www.cnblogs.com/yilangcode/p/15777129.html),在文章的后面，有相同的异常报错，但是哥们直接换了nginx来转发了，不走gateway了。囧。

![image-20231122173702039](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202311221737109.png)



- 升级版本试试

  看了下gateway用的版本`3.1.1`，心想如果升级下就能成功了呢。于是本地搞了demo，把`spring boot`,`spring cloud`以及`jdk`咔咔一顿升级，最后用了

  ``` log
  JDK：17；spring-boot-starter-parent：3.1.5 ；spring-cloud-dependencies：2022.0.4；spring-cloud-starter-gateway：4.0.7
  ```

  期待着会有奇迹发生。然而并没有。接着出现了`java.lang.ClassCastException: class org.apache.catalina.connector.ResponseFacade cannot be cast to class reactor.netty.http.server.HttpServerResponse (org.apache.catalina.connector.ResponseFacade and reactor.netty.http.server.HttpServerResponse are in unnamed module of loader 'app')` 这个异常。

 

   这个异常好找，在github中找到了答案[websocket connction question](https://github.com/spring-cloud/spring-cloud-gateway/issues/2080) ,需要加入以下bean的配置：

   ``` java
       @Bean
       @Primary
       public RequestUpgradeStrategy requestUpgradeStrategy() {
           return new TomcatRequestUpgradeStrategy();
       }
   
       @Bean
       @Primary
       WebSocketClient tomcatWebSocketClient() {
           return new TomcatWebSocketClient();
       }
   ```



加了重新启动，发现好使了，能成功链接访问了。

![image-20231122175508441](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202311221755488.png)



#### nacos负载均衡

在仔细看看之前的gateway配置，是没进行负载均衡配置的，直接固定转给某个websocket服务。环境上是需要负载均衡配置的，因此改个配置试试。

``` yaml
      routes:
        - id: wsroute
          uri: lb:ws://websocket-server
          predicates:
            - Path=/websocket-server/ws/**
            - Header=Connection,Upgrade
          filters:
            - StripPrefix=1
```



当启动后，发现又不好使了，报了503异常，原来是忘了加入feign相关组件依赖：

``` xml
        <!--fegin组件-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!-- Feign Client for loadBalancing -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-loadbalancer</artifactId>
        </dependency>
```

