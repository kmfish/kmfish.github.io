---
title: 一个有趣的需求实现过程：人脸表情包制作
date: 2017-02-25 00:48:12
tags:
---

# 需求
产品有个需求是使用用户上传的照片，自动生成目前比较火的表情包，如下图：
![aaa](https://pic1.zhimg.com/f951e904f83706764c9e659be74f55dc_b.png)
<!--more-->

# 技术点
1. 人脸识别，这是购买别人的SDK实现的啦 。
2. 图像的处理，抠图、图像变换、裁剪、叠加到背景等。

# 实现过程
简单分析了下，要实现这个效果，也有几个关键点要处理好：
- 图像灰度、亮度、对比度的调整
- 脸部区域扣取和边缘模糊

当然，如果要进一步优化下去，也有很多的细节了：
- 人脸方向和背景不一致，则需要调整
- 人脸背景色和表情背景色不一致的情况，要调整；
- 人脸图像大小和位置也需要调整
。。。  
先从基础的这几个点介绍下吧。

## 图像效果处理
这里我使用了GPUImage的滤镜，可以很方便的对原图进行处理。
GPUImageGrayscaleFilter GPUImageContrastFilter GPUImageBrightnessFilter 组合使用这三个滤镜，完成灰度、对比度、亮度的调整即可。

## 脸部区域扣取和边缘模糊
1.  计算脸部区域合适的Path
根据识别SDK给出的人脸五官部位的特征点，来构造一个脸部五官区域的Path。这个Path的计算也经历了几个尝试，主要以两眉和嘴的坐标点来计算，最后的方案是 左眉最左边A，嘴唇下部B，右眉最右边C，从A出发，AB、BC、CA 顺序画了三条贝塞尔曲线，来达到有弧度的边缘效果。

2. 扣取该Path区域
一开始用了canvas.clipPath 来抠图，但发现边缘有锯齿，上网查资料得知，clipPath不能添加抗锯齿效果了，需要采用Paint 配置PorterDuffXfermode 的方式来实现。原理就是，先drawPath画一张Path区域大小的图A，然后再drawBitmap把A画到原图上去，采用PorterDuff.Mode.DST_IN 的模式即可仅保留住path的像素，去掉其余像素。关于PorterDuff模式的使用，也是有点小坑，可以参考下[这篇blog](http://blog.csdn.net/wingichoy/article/details/50534175)。

3. 边缘模糊效果
如果clip的边缘没有模糊效果，那么融合到背景上去是很生硬的，所以需要添加模糊处理。在扣取这步drawPath的时候采用
```java
paint.setMaskFilter(new BlurMaskFilter(EDGE_BLUR, BlurMaskFilter.Blur.NORMAL));
```
即可在path路径上增加模糊效果了。

## 裁剪合适的大小
经过上述步骤后，已经得到了一个仅保留面部区域像素的bitmap（其他区域为透明），此时需要进行一次裁剪，得到一个仅面部大小的图片。在一番尝试后，最后基于path的bounds Rect，再向周围扩大一圈，来计算剪切的rect。这样可以保留住path周围的模糊边缘，才能达到更好的融合效果。

# 后续优化
就这个功能来说，目前的效果也只是demo程度了。后续持续优化的话，可能就需要考虑人脸角度，原片亮度，背景
