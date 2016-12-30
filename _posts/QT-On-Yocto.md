---
published: false
---


### Ubuntu env

GCC-6 has issue to build the host tools for Yocto

``` bash
$sudo apt-get install gcc-5-base gcc-5-multilib
```

### Yocto build

Be careful, that the ${DISTRO} should be set to "fsl-imx-xwayland", not "fsl-imx-wayland", otherwise you can not get the qtwayland plugin and other components installed. Definitely you can change the conf/local.conf to add the qt plugin to "fsl-imx-wayland" DISTRO.

``` bash
$DISTRO=fsl-imx-xwayland MACHINE=imx6ulevk source fsl-setup-release.sh -b build
$bitbake fsl-image-qt5
```
