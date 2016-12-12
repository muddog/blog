---
title: 玩转mbedOS
date: {}
tags:
  - IoT
  - RTOS
  - mbedOS
  - mbed
published: true
---

# 引子 #

mbedOS是ARM自己打造、主打IoT的一整套软件解决方案，是一个针对ARM CortexM系列处理器的嵌入式开源生态。详细的一些介绍可以参见我的另一篇文章[《IoT的那些操作系统》](http://muddog.pub/2016/11/12/IoT-RTOS/)
从日常的工作、及和其他支持部门的交流来看，mbedOS现在还处于成长和推广阶段，在国内正真使用的客户貌似还很少，国外确实有比较牛X的客户指名道姓需要NXP的硬件来支持mbedOS。本人还是比较看好mbed，毕竟亲爹是ARM，而且提供BLE，802.15.4，6LowPAN，Thread，驱动框架及RTOS等等丰富的软件解决方案。所以么，不要等客户培养起来了再去研究，早起的鸟儿有虫吃。

# 搭建环境 #

mbedOS支持三种开发工具：

1.在线IDE
2.mbed CLI控制台
3.第三方开发工具，如IAR，MDK

在线IDE编译很方便快捷，但没有调试功能。第三方的IDE都是可视化的，也没啥好介绍的。这里注重会来介绍mbed-cli，官方的介绍是它的作用贯穿于整个mbed工作流：代码仓库版本控制、依赖管理、代码发布、从其他地方获取代码、调用编译系统及其他。本文会介绍如何利用mbed-cli来：项目导入、下载、编译、测试等等。
可以这么说，mbed-cli和Android的repo有些类似。用过repo，那么你会很快熟悉mbed-cli。


## mbed-cli安装及配置 ##

无论你是用Windows还是Linux的主机，先准备好安装这四个工具：

- Python - mbed CLI 是用Python写的，并且在 version 2.7.11 上做过完整测试，不兼容Python3.x.
- [Git](https://git-scm.com/) - version 1.9.5 or later
- [Mercurial](https://www.mercurial-scm.org/) - version 2.2.2 or later
- [GNU ARM](https://launchpad.net/gcc-arm-embedded) - ARM GCC交叉编译工具

如果你是跑在Windows上，建议使用Git-Bash。Cygwin在后期编译的时候arm-gcc会有路径错误问题。如果在Linux下，以上这些Git，Mercurial，GNUARM链接都是废话，直接仓库里下载吧。

通过PyPi来安装mbed-cli：
``` bash
$ pip install mbed-cli
```

配置下GCC的路径（你安装GNU ARM的路径）：
``` bash
$ mbed config --global ARM_PATH "C:\Program Files\ARM"
```

<!-- more -->

## mbed-cli常用命令 ##

``` bash
Commands:

    new        Create new mbed program or library
    import     Import program from URL
    add        Add library from URL
    remove     Remove library
    deploy     Find and add missing libraries
    publish    Publish program or library
    update     Update to branch, tag, revision or latest
    sync       Synchronize library references
    ls         View dependency tree
    status     Show version control status

    compile    Compile code using the mbed build tools
    test       Find, build and run tests
    export     Generate an IDE project
    detect     Detect connected mbed targets/boards

    config     Tool configuration
    target     Set or get default target
    toolchain  Set or get default toolchain

```





# 编译系统及配置 #

## 导入项目 ##

## 编译 ##

## 配置项目 ##

# 内核分析 #

# 创建自己的App #

## 编译App ##
## 调试App ##


# 总结 #
