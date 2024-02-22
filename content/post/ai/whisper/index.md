---
title: "Whisper语音转文字"
description: 
date: 2024-02-22T15:44:27+08:00
image: post/ai/igor-omilaev-eGGFZ5X2LnA-unsplash.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - AI
tags:
    - AI
    - Whisper
---

最近有个需求，想把语音转成文字，网上搜了搜，能用的免费的没多少，某飞的居然要收88大洋，太贵了，买不起根本买不起。后面寻着了个免费的方法，采用whisper进行语音文字转换。

#### 薅个云主机

进入google云盘(https://drive.google.com/drive/home),新建-更多-Google Colaboratory.

![image-20240222155213774](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202402221552821.png)



接着修改代码运行时类型，更改相应配置：

![image-20240222155511367](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202402221555405.png)

这里选择的运行时：Python3 ,硬件加速器 T4 GPU

![image-20240222155552497](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202402221555542.png)

然后保存之，右上角连接到对应的主机。

![image-20240222155817964](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202402221558994.png)

#### 安装whisper及配套组件

- 在代码栏中输入以下内容，并运行

  ``` python
  !pip install git+https://github.com/openai/whisper.git
  !sudo apt update && sudo apt install ffmpeg
  ```

  

![image-20240222160037099](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202402221600186.png)



上传要转文字的音频或视频文件：

![image-20240222160152788](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202402221601822.png)

接着执行转换命令：

``` shell
!whisper "meeting_01.mp4" --model large-v3
```



根据文件大小，会等待一段时间。最终成功后会在文件夹下生成转换出来的 srt文件，txt文件，json文件等，识别率也还不错。
