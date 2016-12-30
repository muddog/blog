---
published: false
---

### Preparation

- PC Host: x86_64 Ubuntu-16.10
- Target: i.mx6ul 14x14 EVK with LCD pannel
- Mico-SD card
- Ethernet cable & USB cable for console & Power adapter

### Install QTCreator & QT5.x

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

### Ubuntu env

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

### Yocto build

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
