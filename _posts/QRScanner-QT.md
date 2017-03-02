---
published: false
date: {}
tags:
  - Linux
  - QT
  - Gstreamer
title: QT二维码扫码应用（i.MX6UL）
---
# 引子

去年年底还在裸写寄存器编程，今年就开始回归到Linux，还玩起了QT。这个跨度真是大啊。第一任务是在i.MX6UL/ULL上用Yocto做一个参考的带QT开发环境的镜像，并且在该环境下写个二维码扫码应用。虽然一开始接触Yocto，还是蛮陌生的，花了点时间，中间关于QCamera的支持上还绕了点弯路。但总算顺利搞定。

i.MX6UL属于i.MX系列中相对低端的AP，528Mhz的主频（6ULL会有1GHz），摄像头接口，2D硬件加速（例如CSC/FLIP/ROTATION），并且带有LCD显示。可以广泛应用于工业市场，特别是工控HMI，做简单的人机界面。HMI不需要3D渲染加速，但需要有一个能够简单上手的GUI开发环境，所以Linux + QT相当的适合。并且随着线上、线下互动的增强，移动支付，产线物流物品的追溯和管理，二维码扫码的应用也使用的越来越广泛。所以QT加上一个二维码扫码的应用对于在工业市场上的推广比较有利。

之前的博文也提到了如何去搭建一个完整的QT开发环境，以及ROOTFS。也可以参见这里：[https://community.nxp.com/docs/DOC-333580](https://community.nxp.com/docs/DOC-333580)
 
# 开源代码

目前的这个二维码扫码应用程序（QRScanner），是基于QT5.6 + QZXing开发的，可以跑在i.MX6UL EVK板子上。测试用的是UVC的摄像头（640x480分辨率），480x272的标配LCD。

代码开源在我的Github上：[https://github.com/muddog/QRScanner](https://github.com/muddog/QRScanner) 

# 截图
![](https://community.nxp.com/servlet/JiveServlet/downloadImage/102-333910-1-177984/screenshot-qrscanner.png)

# 应用的实现

## 思路
做这种多媒体的应用，特别是和摄像头、视频显示相关的，你肯定会第一个想到使用Gstreamer。没错！其实一个简单的命令就可以验证你的思路。例如在6UL上做摄像头的预览(viewfinder)，并且用PXP做YUV->RGB的转换，及Scale：
``` bash
$ gst-launch-1.0 v4l2src device=/dev/video1 ! video/x-raw,format=YUY2,width=640,height=320 ! imxvideoconvert_pxp ! video/x-raw,format=RGB16 ! waylandsink
```

CPU的loading大概在20-30%，但如果不用PXP做加速，那么loading大概在50-60%。所以肯定需要加速啦。但还有个问题，既然我们用了QT，QT是否有可以直接用的组件呢？查了下Qtmultimedia，有个QCamera类，底下的实现在Linux上是基于Gstreamer的，看似很美好啊，我们可以直接用QCamera么？答案是可以，但性能太差，待会儿会讲到。所以绕了这个弯路后，终于回到了原点，还是老老实实利用GStreamer的API来实现吧。Appsink可以将获取的摄像头桢数据传给应用程序，所以，我们的目标就是建立类似上面的pipeline，并把waylandsink替换成appsink。

## 代码分析

代码很简单，实现几个类：
-**ScannerQWidgetSink**
利用Gst appsink实现一个做视频预览的sink。初始化Gstreamer pipeline，创建一个定时器，每隔一段时间（50ms）将从摄像头获取的桢，用appsink API拷贝到内部的buffer，并且通知做预览的Widget更新界面（ViewfinderWidget redraw）

-**ViewfinderWidget**
继承自QWidget，实现将appsink取来的桢打包成一个QImage实例，并且使用QPainter画到自己的surface上。

-**MainWindow**
主界面类，没有边框和标题栏，全屏。在界面左半边实例化了ViewfinderWidget，右边实例化一个QLabel来显示扫码结果。并且利用动画在预览Widget上绘制红色扫码线动画。

-**DecoderThread**
独立线程。等待ScannerQWidgetSink每0.5s释放一帧图像资源，然后将该数据拷贝到自己的buffer里，送给QZXing引擎做二维码解析。最后将结果显示在QLabel实例上。
 
## 为什么不用QCamera

QCamera用的是Gstreamer bad plugins里的camerabin2，这个BIN消耗的资源特别大，对于Preview，Image Capture，Video Recording各有三套不同的pipeline，还有一波proxy PAD，而且PAD都用的是纯软件的。光pipeline negoation就要花半天，更别说跑起来的速度，在没有PXP的加速下Preview CPU就要耗掉60-70%。我试图在Camerabin2的viewfinder pipeline里用imxvideoconvert_pxp替代videoconvert来加速CSC, Scale，但是始终会出错整个pipeline停止。这个问题找了我将近1个多礼拜。后来发现Qtmultimedia在处理非xcb的platform时候，不会用到Gstreamer内的sink来做显示加速。而使用自己写的qgstvideorendersink类来实现sink，可该sink有bug，会和需要render的widget不同步，导致自己报错。找了个workaround后，CPU loading降了下来，但后面image capture问题又来了，取一帧数据速度实在太慢，好几秒，根本无法接受。所以干脆放弃，还是直接自己建gst pipeline得了。

