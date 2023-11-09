---
title: "Spring boot 调用Grpc"
description: 
date: 2023-11-09T11:06:11+08:00
image: steve-busch-b_nBSjoGtrU-unsplash.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - grpc
tags:
    - java
    - grpc
---
### 起因
最近因工作需要，需要调用grpc服务的一个接口，以前也没接触过，接口提供方呢就扔了个proto文件过来，没法，在网上到处翻翻找找，说的都比较杂乱，缺少部分细节，就简单写个文章记录下吧。

### 具体细节
- 引入proto文件
  在工程`\src\main`目录下新建proto目录，将服务提供方提供的proto文件放入该目录。
  
  ![image-20231109111958425](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202311091119459.png)

- pom.xml文件引入依赖及maven插件：

  ``` xml
          <!-- grpc client -->
          <dependency>
              <groupId>net.devh</groupId>
              <artifactId>grpc-client-spring-boot-starter</artifactId>
              <version>2.14.0.RELEASE</version>
          </dependency>
  ```

  maven插件：

  ``` xml
  		<extensions>
              <!-- os-maven-plugin 插件，从 OS 系统中获取参数 -->
              <extension>
                  <groupId>kr.motd.maven</groupId>
                  <artifactId>os-maven-plugin</artifactId>
                  <version>1.5.0.Final</version>
              </extension>
          </extensions>
          <plugins>
              <plugin>
                   <!-- protobuf-maven-plugin 插件，通过 protobuf 文件，生成 Service 和 Message 类 -->
              <plugin>
                  <groupId>org.xolstice.maven.plugins</groupId>
                  <artifactId>protobuf-maven-plugin</artifactId>
                  <version>0.5.1</version>
                  <configuration>
                      <pluginId>grpc-java</pluginId>
                      <protocArtifact>com.google.protobuf:protoc:3.5.1:exe:${os.detected.classifier}
                      </protocArtifact>
                      <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.51.0:exe:${os.detected.classifier}
                      </pluginArtifact>
                  </configuration>
                  <executions>
                      <execution>
                          <goals>
                              <goal>compile</goal>
                              <goal>compile-custom</goal>
                          </goals>
                      </execution>
                  </executions>
  ```

  

- `application.yaml`中添加grpc配置：

  ``` yaml
  grpc:
    client:
      grpc-client:
        address: 'static://${grpc服务地址}'
        enableKeepAlive: true
        keepAliveWithoutCalls: true
        negotiationType: plaintext
  ```

  

- 跑mvn compile任务，将protobuf文件生成对应的Java代码

- 在业务代码中调用grpc服务：

  ``` java
  	@GrpcClient("grpc-client")
      private xxGrpc.xxBlockingStub xxStub;
  
      public boolean sync(){
          Request.SyncRequest request = Request.SyncRequest.newBuilder().setType("sync").build();
          Response.CommonResponse commonResponse = xxStub.syncRulesHandler(request);
          return commonResponse.getCode()==0;
      }
  ```

  

- 验证grpc服务：

  如果只是想单纯验证下grpc服务是否正常，可以用postman进行操作。

  - 选择请求类型grpc：

    ![image-20231109113945659](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202311091140227.png)

  - 导入protobuf文件：
  
    ![image-20231109114236351](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202311091142390.png)
  
  - 接着就可以输入grpc服务地址及对应参数进行测试了。