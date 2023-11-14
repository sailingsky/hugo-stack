---
title: "Idea构建打包上传Docker镜像"
description: 
date: 2023-11-14T11:16:14+08:00
image: 5-maniere-deplacer-conteneur-docker-vers-autre-machine.jpeg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - docker
tags:
    - docker
---
目前很多应用开发部署都是基于Docker容器化，为了简化本地开发测试，可以在idea中进行相关配置，即可实现本地打包部署高效率。

## 前置条件
- 装有Docker的服务器
- Idea

## Docker服务器配置
-  修改`/usr/lib/systemd/system/docker.service`配置：
``` bash
vi /usr/lib/systemd/system/docker.service
```

在配置项`ExecStart`中加入` -H tcp://0.0.0.0:10086`：
![](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202305022055521.png)

- 配置完成后记得重启docker

  ``` bash
  systemctl daemon-reload 
  systemctl restart docker 
  ```

- 顺便记得确认防火墙，相关端口记得开放：

  ``` bash
  firewall-cmd --zone=public --add-port=10086/tcp --permanent
  
  firewall-cmd --reload
  #查看开放的端口
  firewall-cmd --list-all
  ```

##  idea配置:

- `settings->Build->Docker`新增docker配置，输入docker服务所在ip及之前配置的端口

![image-20230502211249778](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202305022112836.png)

- 配置Docker Registry,这步作用是方便拉取镜像打包时所依赖的其他镜像，如不配置国内镜像源，会容易出现镜像拉取失败：

  ![image-20230502211623623](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202305022116673.png)

## 建立示例工程测试：

- idea中建个springboot工程，此步骤简单略过。

- 修改`pom.xml`文件，加入plugin,`dockerHost`属性配置之前定义好的:

  ``` xml
              <plugin>
                  <groupId>com.spotify</groupId>
                  <artifactId>docker-maven-plugin</artifactId>
                  <version>1.2.2</version>
                  <executions>
                      <execution>
                          <id>build-image</id>
                          <phase>package</phase>
                          <goals>
                              <goal>build</goal>
                          </goals>
                      </execution>
                  </executions>
                  <configuration>
                      <dockerHost>http://192.168.65.131:2375</dockerHost>
                      <imageName>zsj/${project.artifactId}</imageName>
                      <imageTags>
                          <imageTag>${project.version}</imageTag>
                      </imageTags>
                      <forceTags>true</forceTags>
                      <dockerDirectory>${project.basedir}</dockerDirectory>
                      <resources>
                          <resource>
                              <targetPath>/</targetPath>
                              <directory>${project.build.directory}</directory>
                              <include>${project.build.finalName}.jar</include>
                          </resource>
                      </resources>
                  </configuration>
              </plugin>
  ```

  

- 工程根目录新增Dockerfile:

  ``` dockerfile
  FROM openjdk:8-jdk-alpine
  ARG JAR_FILE=target/*.jar
  COPY ${JAR_FILE} app.jar
  ENTRYPOINT ["java","-jar","/app.jar"]
  ```

  

- 运行maven的package任务：

  ![image-20230502212207557](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202305022122604.png)

成功后可在控制台看到如下输出：

![image-20230502212246029](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202305022122068.png)



- 登录docker所在服务器，查看镜像是否打包上传成功：

  ![image-20230502212343565](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202305022123592.png)