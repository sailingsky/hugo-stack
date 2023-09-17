---
title: "Build Blog"
description: hugo建博客
date: 2023-09-17T18:07:30+08:00
image: cover.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - 建站
tags:
    - 建站
    - stack
    - hugo
---
### 下载安装hugo
- 去github上下载对应系统的安装包：[hugo安装包](https://github.com/gohugoio/hugo/releases/tag/v0.118.2)。由于我是windows系统，那就下载windows版本。
- 下载完成后，解压将安装文件放至到某个目录。修改环境变量path,添加刚才hugo文件所在路径。
-  命令行验证是否本地安装成功：
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202309170952606.png)

### 创建Github仓库
#### 创建博客源文件内容仓库：

名字随意取
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202309170955055.png)

#### 创建GitHub Pages仓库：
仓库名称必须使用`<username.github.io>`的格式，`username`为GitHub的用户名:
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202309171000912.png)


~~### 将博客源文件内容仓库拉到本地：

```shell
 git clone https://github.com/username/blog.git
```

~~### 用hugo创建网站：
- 进入刚下的本地仓库文件夹，如`blog`
- 使用命令`hugo new site 网站名称`创建网站。
```bash
cd blog
hugo new site bytecode-blog
```
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202309171011884.png)

~~### 安装和配置hugo主题

进入到之前的`myblog`目录，根据选择的hugo主题安装文件，我这里选择的是`stack`主题：
```shell
git submodule add https://github.com/CaiJimmy/hugo-theme-stack/ themes/hugo-theme-stack
```


~~### 配置主题

将`themes\hugo-theme-stack\exampleSite`里面的`content`文件以及`config.yaml`拷贝覆盖到根目录下


### 更简单的作法，前面删除线的步骤皆可不用！
- 如果确定使用`stack`主题，直接去GitHub下载[stack-starter](https://github.com/CaiJimmy/hugo-theme-stack-starter)工程。
- 修改`config\_default`目录下的`config.toml`文件，将`baseurl`修改为：`https://<github-username>.github.io/`
- 运行`hugo server`命令，即可在本地看到效果。
- 运行`hugo`命令，将生成对应的静态文件。在`public`文件夹下。
- 进入到`public`文件夹，执行以下命令,将静态文件上传到Github：
```bash
 git init -b main

 git remote add origin git@github.com:<github-username>/<github-username>.github.io.git

 git pull --rebase origin main

 git add .

 git commit -m 'message'

 git push origin main
```

### 查看效果：
接着你就可以访问`https://<github-username>.github.io`查看效果了：
![image.png](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202309171759286.png)
