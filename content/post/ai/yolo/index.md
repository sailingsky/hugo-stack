---
title: "YOLO火情识别"
description: 
date: 2026-01-02T16:13:11+08:00
image: post/ai/pexels-ahmetyuksek-35881620.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - AI
tags:
    - AI
    - YOLO
---
# YOLO火情识别

前提：有张N卡😊，更新驱动至最新。

1. 安装cuda:

   1. 下载安装包安装

      https://developer.nvidia.com/cuda-downloads?target_os=Windows&target_arch=x86_64&target_version=11&target_type=exe_local

   2. 安装后验证安装是否成功

      ``` shel
      nvcc -V
      ```

      ![image-20260130153255413](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202601301532461.png)

2. 搞个数据集，可以自己打标搞或找个别人搞好的

   1. 这里demo找了个开源的，下载到本地：

      https://universe.roboflow.com/never-gonna-give-you-up/123-wdvvi/dataset/2

      

3. 安装yolo依赖：

   ``` shell
   pip install ultralytics
   ```
   下载模型文件,我选的`yolo26m.pt`:
   https://github.com/ultralytics/assets/releases

4. 运行&调试：

   1. 试运行：

      ``` shell
      yolo task=detect mode=train model=model/yolo26m.pt data=dataset/data.yaml epochs=100 imgsz=640 device=0 
      ```

      结果出现错误:

      ``` log
      torch.cuda.is_available(): False
      torch.cuda.device_count(): 0
      os.environ['CUDA_VISIBLE_DEVICES']: None
      See https://pytorch.org/get-started/locally/ for up-to-date torch install instructions if no CUDA devices are seen by torch.
      ```

   2. 卸载cpu版本的Torch:

      ``` shell
      pip uninstall torch torchvision torchaudio -y
      ```

   3. 安装支持cuda版本依赖:

      ``` shell
      pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
      ```

   4. 检查CUDA 是否可用:
   
      ``` python
      import torch
      print(f"CUDA 是否可用: {torch.cuda.is_available()}")
      print(f"显卡名称: {torch.cuda.get_device_name(0)}")
      ```
   
   5. 可用后，继续运行4.1的模型训练，然后就漫长的等待,我等了一天.（或者可以用现成的别人训练好的：https://www.kaggle.com/code/bertnardomariouskono/smoke-fire-detection-yolo/notebook）
   
      ![795e06911c25fc9f6fe4eb9d04871eb0](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202602021548556.jpg)
   
   6. 训练完成后，会在对应目录下生成个`best.pt`文件，我们用这个文件进行目标图像检测:
   
      ``` python
      from ultralytics import YOLO
      
      model = YOLO('runs/detect/train3/weights/best.pt')
      #当检测图片或视频流时，建议stream=True
      results = model.predict(source='images/',conf=0.3,stream=False)
      
      for result in results:
          fire_count = 0
          max_area_ratio = 0
          img_area = result.orig_shape[0] * result.orig_shape[1]
          print(f"\n>>> 正在分析: {result.path}")
          print(f"检测到的框数量: {len(result.boxes)}")
          # 显示检测后的图像
          result.show()
          for box in result.boxes:
              cls_id = int(box.cls[0])
              label = model.names[cls_id]
              confidence = box.conf
              if label == 'smoke':
                  print(f"发现烟雾！")
              if label == 'fire':
                  print(f"【高风险：发现火源】")
      ```
   
      

YOLO的更详细的用法可参考:https://docs.ultralytics.com/modes/predict/。当然这篇文章只是简单的识别下烟雾和火源，更深层次的扩展还可以继续深挖。

5. 效果展示:

   原图：

   ![image-20260202163106093](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202602021631679.png)

YOLO检测标记出的图：

![tmpckhm8rl0](https://wechapter.oss-cn-hangzhou.aliyuncs.com/wechat/image202602021632580.PNG)

