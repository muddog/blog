---
published: true
date: {}
tags:
  - QT
  - Linux
  - i.MX6
  - Yocto
title: Enable QT developement for i.MX6UL
---

Qt is a cross-platform application framework that is widely used for developing application software that can be run on various software and hardware platforms with little or no change in the underlying codebase, while still being a native application with native capabilities and speed. It is running in TV, Car, HMI, 3D Printer, almost covering all of the embedded areas. This document introduces how to setup the development environment to build Qt application, deploy to NXP's i.MX6UL EVK board, and run application on the Linux BSP.

## Hardware Requirement

- PC Host: x86_64 Ubuntu-16.10
- Target: i.mx6ul 14x14 EVK with LCD pannel
- Mico-SD card
- Ethernet cable & USB cable for console & Power adapter

## QT SDK

QT one ecosystem provide SDK to build powerful, connected and beautiful applications that run on any screen and across any platform. Here for embedded development, we need the following tools:
1. Qt Creator IDE
2. Qt5.6 source code
3. HOST build toolchain
4. TARGET build toolchain

Qt Creator is a cross-platform C++, JavaScript and QML integrated development environment which is part of the SDK for the Qt GUI Application development framework. It includes a visual debugger and an integrated GUI layout and forms designer. It can run on both Windows, MacOS and Linux, developer create a Qt application on PC HOST and debug the UI on HOST build with HOST toolchain firstly. When debug completed, developer could use TARGET toolchain to build the application, and deploy the binary to embedded system like i.MX6UL linux.

The HOST toolchain would be downloaded with SDK, but the TARGET toolchain is downloaded by Yocto introduced below.

### Setup Ubuntu

The default GCC-6 on Ubuntu 16.10 has issue to build the lzop tools for Yocto, install the GCC-5 and create gcc/g++ link to GCC-5 under /usr/bin.

``` bash
$ sudo apt-get install gcc-5-base gcc-5-multilib
```
Install required packages:
``` bash
$ sudo apt-get install build-essential # for g++
$ sudo apt-get install libfontconfig1  # for font config
$ sudo apt-get install libglu1-mesa-dev # for HOST OpenGL
```

### Install Qt SDK

Online or offline installer for Linux version can be found on:
https://www.qt.io/download-open-source/#section-2
For example, we download a online one here: http://download.qt.io/official_releases/online_installers/qt-unified-linux-x64-online.run

Grant the executable permission to this binary, and run it.
``` bash
$ chmod +x qt-unified-linux-x64-2.0.4-online.run
$ ./qt-unified-linux-x64-2.0.4-online.run
```

If you do not have a QT account, just create one. Otherwise login with your account, then "Next". When the installer asking you to chose the components, please chose the **"Qt 5.6"** which consistent with the Yocto i.MX target QT version with the **"Toolchain"** and **"Source Components"** selected. Also the **"Qt Creator"** is mandatory like below:
![](http://ohx9w4r3g.bkt.clouddn.com/blog/qt/qtcreator-install.png)

The **"Toolchain"** is used for HOST build for applications, which you can debug your application on HOST before deploy to target. The **"Source Components"** contains the opensource code of QT and many examples.

After all the packages downloaded and installed, you can run the Qt Creator by:
``` bash
$ <Qt install dir>/Tools/QtCreator/bin/qtcreator
```

You would be able to open the examples in the Qt5.x source code and quickly use the HOST toolchain to build and run a example on PC.


## i.MX6UL Yocto BSP

### Yocto build

Get the repo, this tool can be download from the following sites:
- http://php.webtutor.pl/en/wp-content/uploads/2011/09/repo
- https://dl-ssl.google.com/dl/googlesource/git-repo/repo
- http://android.git.kernel.org/repo

Please chose the workable one, for China user, the following is workable:

``` bash
$ curl http://php.webtutor.pl/en/wp-content/uploads/2011/09/repo > ~/repo
$ chmod a+x ~/repo
$ export PATH=$PATH:~/
```

**Download FSL i.MX Yocto BSP**

``` bash
$ repo init -u git://git.freescale.com/imx/fsl-arm-yocto-bsp.git -b imx-4.1-krogo
$ repo sync
```

**Setup environment**

Be careful, that the ${DISTRO} should be set to "fsl-imx-xwayland", not "fsl-imx-wayland", otherwise you can not get the qtwayland plugin and other components installed. Definitely you can change the conf/local.conf to add the qt plugin to "fsl-imx-wayland" DISTRO.

``` bash
$ DISTRO=fsl-imx-xwayland MACHINE=imx6ulevk source fsl-setup-release.sh -b build
```

**Add SFTP support**

Qt Creator use the SFTP protocol to upload the target image file, so we have to install ssh-server-openssh package instead of dropbear by change the .bb file:

``` bash
project sources/meta-fsl-bsp-release/
diff --git a/imx/meta-sdk/recipes-fsl/images/fsl-image-validation-imx.bb b/imx/meta-sdk/recipes-fsl/images/fsl-image-validation-imx.bb
index 8343223..9989f83 100644
--- a/imx/meta-sdk/recipes-fsl/images/fsl-image-validation-imx.bb
+++ b/imx/meta-sdk/recipes-fsl/images/fsl-image-validation-imx.bb
@@ -20,7 +20,7 @@ IMAGE_FEATURES += " \
     splash \
     nfs-server \
     tools-debug \
-    ssh-server-dropbear \
+    ssh-server-openssh \
     tools-testapps \
     hwcodecs \
     ${@bb.utils.contains('DISTRO_FEATURES', 'wayland', '', \
     

project sources/meta-fsl-bsp-release/
diff --git a/imx/meta-sdk/recipes-fsl/images/fsl-image-qt5-validation-imx.bb b/imx/meta-sdk/recipes-fsl/images/fsl-image-qt5-validation-imx.bb
index 39c7841..2ab231d 100644
--- a/imx/meta-sdk/recipes-fsl/images/fsl-image-qt5-validation-imx.bb
+++ b/imx/meta-sdk/recipes-fsl/images/fsl-image-qt5-validation-imx.bb
@@ -45,4 +45,7 @@ QT5_IMAGE_INSTALL_remove = " packagegroup-qt5-webengine"
 
 IMAGE_INSTALL += " \
 ${QT5_IMAGE_INSTALL} \
+openssh \
+openssh-sftp \
+openssh-sftp-server \
 "

```

**Build rootfs**

``` bash
$ bitbake fsl-image-qt5
```

Build the QT5 target toolchain, which would be used on the QTCreator to build and deploy the QT application:

``` bash
$ bitbake meta-toolchain-qt5
```

### Deploy SDcard image

Deploy the rootfs, and install the cross compile toolchain, cross Qt5 library into HOST used by Qt Creator
``` bash
$ sh tmp/deploy/sdk/fsl-imx-xwayland-glibc-x86_64-meta-toolchain-qt5-cortexa7hf-neon-toolchain-4.1.15-2.0.1.sh
```

You will find the i.MX6UL uboot/zImage/rootfs(ext4)/sdcard image under: build/tmp/deploy/images/imx6ulevk/. Prepare your SD card, and use the following dd command to deploy whole images:
``` bash
$ sudo dd if=build/tmp/deploy/images/imx6ulevk/fsl-image-qt5-imx6ulevk.sdcard of=/dev/<sdcard dev>
```

The Qt5 cross compile tools are installed under: /opt/fsl-imx-xwayland/4.1.15-2.0.1/sysroots/x86_64-pokysdk-linux/
The target configuration files, includes and libs are under opt/fsl-imx-xwayland/4.1.15-2.0.1/sysroots/cortexa7hf-neon-poky-linux-gnueabi/


## Qt Creator configure

Launch Qt Creator, create one project or use the example project and open it.

### Configure the cross toolchain

Open the "Build & Run" settings from [Menu] -> [Tools] -> [Options...].
1. Add target Qt5 qmake (deploy before under /opt/fsl-imx-xwayland/4.1.15-2.0.1/sysroots/x86_64-pokysdk-linux/usr/bin/qt5) in the tab of "Qt versions":

![](http://ohx9w4r3g.bkt.clouddn.com/blog/qt/qt_versions.png)

2. 

### Qt mkspec

Update the qmake.conf file under:**/opt/fsl-imx-xwayland/4.1.15-2.0.1/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/lib/qt5/mkspecs/linux-arm-gnueabi-g++/qmake.conf** to update the toolchain name (arm-poky-linux-gnueabi-), the --sysroot for linker, and the NEON VFP4 hardware float support (as the toolchain is built for hf). Without this update, the linked library can not be found and the link would failed due to different float-abi usage.

``` qmake
@@ -1,5 +1,5 @@
 #
-# qmake configuration for building with arm-linux-gnueabi-g++
+# qmake configuration for building with arm-poky-linux-gnueabi-g++
 #
 
 MAKEFILE_GENERATOR      = UNIX
@@ -11,14 +11,17 @@
 include(../common/g++-unix.conf)
 
 # modifications to g++.conf
-QMAKE_CC                = arm-linux-gnueabi-gcc
-QMAKE_CXX               = arm-linux-gnueabi-g++
-QMAKE_LINK              = arm-linux-gnueabi-g++
-QMAKE_LINK_SHLIB        = arm-linux-gnueabi-g++
+QMAKE_CC                = arm-poky-linux-gnueabi-gcc
+QMAKE_CXX               = arm-poky-linux-gnueabi-g++
+QMAKE_LINK              = arm-poky-linux-gnueabi-g++
+QMAKE_LINK_SHLIB        = arm-poky-linux-gnueabi-g++
+
+QMAKE_LFLAGS += --sysroot=/opt/fsl-imx-xwayland/4.1.15-2.0.1/sysroots/cortexa7hf-neon-poky-linux-gnueabi -mfloat-abi=hard -mfpu=neon-vfpv4
+QMAKE_CXXFLAGS +=  -mfloat-abi=hard -mfpu=neon-vfpv4
 
 # modifications to linux.conf
-QMAKE_AR                = arm-linux-gnueabi-ar cqs
-QMAKE_OBJCOPY           = arm-linux-gnueabi-objcopy
-QMAKE_NM                = arm-linux-gnueabi-nm -P
-QMAKE_STRIP             = arm-linux-gnueabi-strip
+QMAKE_AR                = arm-poky-linux-gnueabi-ar cqs
+QMAKE_OBJCOPY           = arm-poky-linux-gnueabi-objcopy
+QMAKE_NM                = arm-poky-linux-gnueabi-nm -P
+QMAKE_STRIP             = arm-poky-linux-gnueabi-strip
 load(qt_config)

```

### Configure the remote device

Add one "Generic Linux" device for i.MX6UL EVK board. Input the correct IP address, SSH port and username. Click the "Test" button to test the connection between PC and EVK board.

![](http://ohx9w4r3g.bkt.clouddn.com/blog/qt/devices.png)

### .pro for build

In your Qt Creator project, modify the [project].pro
- Change the **target.path** to the target application location you want to download to the board.
- Add three INCLUDEPATH env for target cross compile headers
- Add macro defines for VFP and GL usage

``` Makefile
...
target.path = /home/root/audiodevices
INSTALLS += target
...
INCLUDEPATH += /opt/fsl-imx-xwayland/4.1.15-2.0.1/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/include/c++/5.3.0/
INCLUDEPATH += /opt/fsl-imx-xwayland/4.1.15-2.0.1/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/include/c++/5.3.0/arm-poky-linux-gnueabi/
INCLUDEPATH += /opt/fsl-imx-xwayland/4.1.15-2.0.1/sysroots/cortexa7hf-neon-poky-linux-gnueabi/usr/include/

DEFINES += __ARM_PCS_VFP QT_NO_OPENGL
```

