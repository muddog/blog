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
Implementation
To do camera preview and capture, you must think on the gstreamer first, which is easy use and has the acceleration pads which implemented by NXP for i.MX6UL. Yes, it's very easy for you to enable the preview in console like:
$ gst-launch-1.0 v4l2src device=/dev/video1 ! video/x-raw,format=YUY2,width=640,height=320 ! imxvideoconvert_pxp ! video/x-raw,format=RGB16 ! waylandsink
It works under the i.MX6UL EVK, with PXP IP to do color space convert from YUY2 -> RGB16 acceleration, also the potential scaling of the image. The CPU loading of this is about 20-30%, but if you use the component of "videoconvert" to replace the "imxvideoconvert_pxp", we do CSC and scale by CPU, then the loading would increase to 50-60%. The "/dev/video1" is the device node for UVC camera, it may different in your environment.
So our target is clear, create such pipeline (with PXP acceleration) in the QT application, and use a appsink to get preview images, do simple "sink" to one QWidget by drawing this image on the widget surface for preview (say every 50ms for 20fps). Then in other thread, we fetch the preview buffer in a fixed frequency (like every 0.5s), then feed it into the ZXing engine to decode the strings inside this image.
 
Here are the class created inside the source code:
ScannerQWidgetSink
It act as a gstreamer sink for preview rendering. Init the pipeline, create a timer with timeout every 50ms. In the timer handler, we use appsink to copy the camera buffer from gstreamer, and tell the ViewfinderWidget to do update (re-draw event).
ViewfinderWidget
This class inherit from the QWidget, which draw the preview buffer as a QImage onto it's own surface by using QPainter. The QImage is created at the very begining with the image buffer created by the ScannerQWidgetSink. Because QImage itself does not maintain the image buffer, so the buffer must be alive during it's usage. So we keep this buffer during the ScannerQWidgetSink life cycle, copy the appsink buffer from pipeline to it for preview.
MainWindow
Create main window, which does not have title bar and border. Start any animation for the red line scan bar. Create instance of DecoderThread and ScannerQWidgetSink. Setup and start them.
DecoderThread
A infinite loop, to wait for a available buffer released by the ScannerQWidgetSink every 0.5s. Copy the buffer data to it's own buffer (imgData) to avoid any change to the buffer by sink when doing decoding. Then feed this copy of buffer into ZXing engine to get decoder result. Then show on the QLabel.
 
Screenshot under wayland (weston) desktop:

 
 
Customize
Camera instance
Now I use the UVC camera which pluged in the USB host, which device node is /dev/video1. If you want to use CSI or other device, please change the construction parameters for ScannerQWidgetSink():
sink = new ScannerQWidgetSink(ui->widget, QString("v4l2src device=/dev/video1"));
Image resolution captured and review
Change the static member value of ScannerQWidgetSink class:
uint ScannerQWidgetSink::CAPTURE_HEIGHT = 480;
uint ScannerQWidgetSink::CAPTURE_WIDTH = 640;
Preview fps and decoding frequency
Find the "framerate=20/1" strings in the ScannerQWidgetSink::GstPipelineInit(), change to your fps. You also have to change the renderTimer start timeout value in the ::StartRender(). The decoding frequency is determined by renderCnt, which determine after how many preview frames showed to feed the decoder.
Main window size
It's fixed size of main window, you have to change the mainwindow.ui. It's easy to do in the QtCreate Designer.