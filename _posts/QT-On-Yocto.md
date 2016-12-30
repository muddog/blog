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


## Environment

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

GCC-6 has issue to build the host tools for Yocto

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

Download FSL i.MX Yocto BSP:
``` bash
$ repo init -u git://git.freescale.com/imx/fsl-arm-yocto-bsp.git -b imx-4.1-krogo
$ repo sync
```

Be careful, that the ${DISTRO} should be set to "fsl-imx-xwayland", not "fsl-imx-wayland", otherwise you can not get the qtwayland plugin and other components installed. Definitely you can change the conf/local.conf to add the qt plugin to "fsl-imx-wayland" DISTRO.

``` bash
$ DISTRO=fsl-imx-xwayland MACHINE=imx6ulevk source fsl-setup-release.sh -b build
$ bitbake fsl-image-qt5
```

Then you will have the rootfs ready under build/tmp/sysrootfs/imx6ulevk/

Build the QT5 target toolchain, which would be used on the QTCreator to build and deploy the QT application:

``` bash
$ bitbake meta-toolchain-qt5
```
