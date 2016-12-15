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

## 开发板 ##

mbedOS的例子大部分都可以直接跑在NXP的FRDM-K64之上。

- mbed官方对板子的[介绍及使用方法](https://developer.mbed.org/platforms/FRDM-K64F/)，
- NXP官网对板子的[介绍](http://www.nxp.com/products/software-and-tools/hardware-development-tools/freedom-development-boards/freedom-development-platform-for-kinetis-k64-k63-and-k24-mcus:FRDM-K64F?fsrch=1&sr=1&pageNum=1)

买板子，虽然mbed和NXP都可以买，但还是淘宝吧，发货快。

## 开发工具 ##

mbedOS支持三种开发工具：
1.在线IDE
2.mbed CLI控制台
3.第三方开发工具，如IAR，MDK

在线IDE编译很方便快捷，但没有调试功能。第三方的IDE都是可视化的，也没啥好介绍的。这里注重会来介绍mbed-cli，官方的介绍是它的作用贯穿于整个mbed工作流：代码仓库版本控制、依赖管理、代码发布、从其他地方获取代码、调用编译系统及其他。本文会介绍如何利用mbed-cli来：项目导入、下载、编译、测试等等。
无论你是用Windows还是Linux的主机，先准备好安装这四个工具：

- Python - mbed CLI 是用Python写的，并且在 version 2.7.11 上做过完整测试，不兼容Python3.x.
- [Git](https://git-scm.com/) - version 1.9.5 or later
- [Mercurial](https://www.mercurial-scm.org/) - version 2.2.2 or later
- [GNU ARM](https://launchpad.net/gcc-arm-embedded) - ARM GCC交叉编译工具

<!-- more -->
如果你是跑在Windows上，建议使用Git-Bash。Cygwin在后期编译的时候arm-gcc会有路径错误问题。如果在Linux下，以上这些Git，Mercurial，GNUARM链接都是废话，直接仓库里下载吧。

通过PyPi来安装mbed-cli：
``` bash
$ pip install mbed-cli
```

配置下GCC的路径（你安装GNU ARM的路径）：
``` bash
$ mbed config --global GCC_ARM_PATH "C:\Program Files\ARM"
```

## 调试工具 ##

既然用了GNU的编译工具，GDB肯定是必不可少的调试工具。但是由于涉及到板子设备端的调试，我们还需要安装pyOCD或者OpenOCD。两者都是OCD - On Chip Debugger。都是将GDBServer协议和CMSIS-DAP的设备端调试机制集成在了一起，可以一边通过GDBServer接受GDB过来的调试请求，另一边将这些请求转换成CMSIS-DAP的SWD指令对设备端进行调试。CMSIS-DAP不多说了，基本你可以认为它是板载的调试器，用户不用购买JLINK那么贵的设备，插上USB线就可以调试板子（FRDM-K64也带了板载的CMSIS-DAP）。

### pyOCD ###
pyOCD是mbed官方提供的调试工具，使用Python开发，框架图如下：
![](https://docs.mbed.com/docs/debugging-on-mbed/en/latest/Debugging/Images/PyOCD1.png)

安装方法如下：

``` bash
$ pip install --pre -U pyocd
```

但是pyOCD有个最大的缺陷，monitor的命令非常少（只支持reset, init, halt），对于调试来说手段很有限。所以这里会比较推荐OpenOCD。

### OpenOCD ###
非常强大的开源OCD，支持的调试接口非常多，不局限于CMSIS-DAP，支持包括Jlink，OSBDM，ULINK，ST-LINK等等。简单的安装方式是下载GNU ARM Eclipse OCD包：https://github.com/gnuarmeclipse/openocd/releases 。如果从OpenOCD官网下载，你还需要编译源代码。


## 其他 ##

- 串口工具装一个，方便调试。Putty, minicom，串口调试助手等都可以。
- mbed serial，Windows串口驱动（[下载页面](https://developer.mbed.org/handbook/Windows-serial-configuration) ），Linux不需要。


# 编译系统及配置 #

mbed-cli和Android的repo有些类似。用过repo，那么你会很快熟悉mbed-cli。
如果你用过Makefile，那么mbed的思路有些不太一样。Makefile里可以规定当前项目的目标是什么，我要编译什么源文件，目标的依赖是什么，相对来说比较通用。mbed则比较特殊，这里简单描述下它对项目的组织方式，编译及依赖管理。

拿mbed-os-example-client（https://github.com/ARMmbed/mbed-os-example-client） 这个项目举个例子，列出主要的文件：

``` bash
main.cpp
mbed_app.json
mbed_client_config.h
mbed-client/
 - mbed_lib.json
 - module.json
 - ...
mbed-client.lib
mbed-os/
 - targets/targets.json
mbed-os.lib
mbedtls_mbed_client_config.h
mcr20a-rf-driver/
mcr20a-rf-driver.lib
README.md
....
```

它有自己的源文件，放在顶层，main.cpp（当然你可以有很多个.cpp，mbed会帮你编译）。然后通过mbed_app.json描述项目的基本信息，应用的配置及目标系统的配置覆盖。怎么理解？我们先贴上mbed_app.json内容（省略了一部分）：

``` json
{
    "config": {
        "network-interface":{
            "help": "options are ETHERNET,WIFI,MESH_LOWPAN_ND,MESH_THREAD",
            "value": "ETHERNET"
        },
        "mesh_radio_type": {
                "help": "options are ATMEL, MCR20",
                "value": "MCR20"
        },
...
    },
    "target_overrides": {
        "*": {
            "target.features_add": ["NANOSTACK", "LOWPAN_ROUTER", "COMMON_PAL"],
            "mbed-mesh-api.6lowpan-nd-channel-page": 0,
            "mbed-mesh-api.6lowpan-nd-channel": 12,
            "mbed-trace.enable": 0
        },
}
```

**"config"** 对象里存放的是应用程序会使用到的配置，这些配置项最终会被mbed-cli转换成C语言里的宏，放到生成的mbed_config.h中，例如**"mesh_radio_type"**和**"network-interface"**，会被转换成：

``` C
#define MBED_CONF_APP_MESH_RADIO_TYPE  MCR20 
#define MBED_CONF_APP_NETWORK_INTERFACE  ETHERNET
```

转换规则很简单，连接符-转换成下划线，小写变大写，加上"MBED_CONF_APP" 前缀。所以在你的应用程序中，可配置项不会像常用的项目那样给用户一个.h文件，定义一些宏，让用户直接修改。而是在源文件中直接使用宏，宏的定义交给.json配置文件去写。这样的好处是通过JSON格式，可配置项变的易读，而且不用用户去创建或者修改.h文件。下面的"target_overrides"比较特殊，其实是针对mbedOS依赖项目（mbed_os/）里的设备端的软硬件配置。基本每个项目都需要用到它去覆盖某些目标设备的配置，毕竟软件还是要跑在板子上的。"target_overrides"可以选择覆盖任意，比如"\*"，或者覆盖某个硬件平台配置，比如"K64"。那么这些被覆盖的配置原始定义在哪里呢？
对于硬件平台，target.打头的配置，可以在mbedOS的targets目录下找到：
> mbed-os/targets/targets.json

K64的配置：
``` json
    "K64F": {
        "supported_form_factors": ["ARDUINO"],
        "core": "Cortex-M4F",
        "supported_toolchains": ["ARM", "GCC_ARM", "IAR"],
        "extra_labels": ["Freescale", "KSDK2_MCUS", "FRDM", "KPSDK_MCUS", "KPSDK_CODE", "MCU_K64F"],
        "is_disk_virtual": true,
        "macros": ["CPU_MK64FN1M0VMD12", "FSL_RTOS_MBED"],
        "inherits": ["Target"],
        "detect_code": ["0240"],
        "device_has": ["ANALOGIN", "ANALOGOUT", "ERROR_RED", "I2C", "I2CSLAVE", "INTERRUPTIN", "LOWPOWERTIMER", "PORTIN", "PORTINOUT", "PORTOUT", "PWMOUT", "RTC", "SERIAL", "SERIAL_FC", "SLEEP", "SPI", "SPISLAVE", "STDIO_MESSAGES", "STORAGE", "TRNG"],
        "features": ["LWIP", "STORAGE"],
        "release_versions": ["2", "5"],
        "device_name": "MK64FN1M0xxx12"
    },

```

"target.features_add"表明除了lwip, storage外，我们还需要使能mbedOS里的nanostack及6lowpan的路由功能。
对于其他mbedOS相关的软件配置，例如mbed-mesh-api.打头的配置，可以在mesh网络的stack路径里找到：
> mbed-os/features/nanostack/FEATURE_NANOSTACK/mbed-mesh-api/mbed_lib.json

这些配置都最终会以宏的形式保存在BUILD/[target]/[toolchain]/mbed_config.h文件中。当然，你选用什么样的硬件target，是在编译器选择的。

项目依赖则通过.lib文件来描述，.lib文件其实是一个txt文件，描述了依赖库的源码git url以及版本信息。mbed-os-example-client依赖于mbed Client客户端的应用（mbed_client.lib），依赖于mbedOS（mbed_os.lib 包括RTOS，驱动，网络协议栈），依赖于MCR20A的RF驱动（mcr20a-rf-driver.lib）。例如其中的mbed_os.lib内容如下：
> https://github.com/ARMmbed/mbed-os/#d5de476f74dd4de27012eb74ede078f6330dfc3fe

\#号后面是commit id，指定版本。对应的目录则是其源代码或者library。

那么对于这些库library来讲，它的编译配置及库的基本信息mbed怎么得知？和应用程序类似，我们看到mbed-client下有两个文件：
- module.json
- mbed_lib.json

第一个文件描述了该库的名字、版本、License、作者等等的基本信息，以及库所依赖的其他库版本。第二个文件则和应用程序的mbed_app.json一样，保存的编译配置。
讲到这里，你应该可以理解mbed的编译配置环境了。接下来可以Hands-on了。

# 动一动手 #

## 导入项目 ##

``` bash
$ mbed import <url>
```
和repo init + sync类似。URL可以是完整的git repo路径，如果只给项目名称，会直接从https://github.com/ARMmbed/ 里的项目：
``` bash
$ mbed import mbed-os-example-client
```
该命令会处理模块之间的依赖关系，会检查mbed_app.json配置及.lib文件。确保所有依赖的项目源文件都被下载。如果你用git clone将mbed-os-example-client克隆下来，那么依赖库不会被同时下载，需要使用下面的命令：
``` bash
$ mbed deploy
```

## 创建新项目 ##
``` bash
$ mbed new my-mbed-app
```
默认会生成并下载mbedOS这个依赖库，如果需要其他库，可以用下面的命令:
``` bash
$ mbed add <git repo url>
```
mbed会自动下载，保存在子目录下，并且帮你生产.lib文件。不需要这个依赖库时，可以使用remove命令删除.lib及源文件。通过以下命令可以查看当前项目的依赖关系：
``` bash
$ mbed ls
my-mbed-app (None)
|- mbed-client (203b74116e2e)
|  |- mbed-client-classic (42f18d977122)
|  `- mbed-client-mbed-tls (7e1b6d815038)
`- mbed-os (d5de476f74dd)

```

## 编译 ##

编译需要指定target及toolchain：

``` bash
$ mbed compile -m K64 -t GCC_ARM
```
指定一次即可，后面可以直接使用compile编译，如果需要clean build，加上-c

## 调试 ##

前面的环境搭建里，安装好的pyOCD作为GDBServer，而GNU ARM Toolchain里的GDB则作为Client用来调试。

### 打开调试选项 ###
首先要把编译器的-g选项打开，将debug symbol编译进elf文件。那么mbedOS里提供了对应的toolchain配置profiles：
``` bash
$ mbed compile -c --profile mbed-os/tools/profiles/debug.json
```
这个debug.json里都是编译器选项，你可以按照自己的要求修改。编译结果：./BUILD/[targe]>/[toolchain]/[project].elf

### 启动GDBServer ###

这里只讲OpenOCD。pyOCD比较简单，但调试用起来很不爽。首先，我们要自己为FRDM-K64写一个openOCD配置脚本，默认的release里没有。首先跑到GNU ARM Eclipse OCD安装目录下，例如：/c/Program Files/GNU ARM Eclipse/OpenOCD/0.10.0-201610281609-dev。在scripts/board/下新建一个frdm-k64.cfg，内容如下：

``` bash
source [find interface/cmsis-dap.cfg]

# increase working area to 16KB
set WORKAREASIZE 0x4000

# chip name
set CHIPNAME MK64FN1M0VLL12

reset_config srst_only

source [find target/kx.cfg]

```

然后启动OpenOCD GDBServer:

``` bash
$ bin/openocd.exe -f scripts/board/frdm-k64.cfg
```
默认绑定本地3333端口


### 调试 ###

链接到GDBServer （3333端口）：
``` bash
$ arm-none-eabi-gdb.exe ./BUILD/K64F/GCC_ARM/mbed-os-example-client.elf
GNU gdb (GNU Tools for ARM Embedded Processors) 7.6.0.20140228-cvs
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=i686-w64-mingw32 --target=arm-none-eabi".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from D:\mcu\iot\mbed-os-example-client\BUILD\K64F\GCC_ARM\mbed-os-example-client.elf...done.

(gdb) target remote localhost:3333

```

然后通过OpenOCD将程序烧入Flash：
``` bash
(gdb) mointor program mbed-os-example-clinet.bin
(gdb) Undefined command: "mointor".  Try "help".
monitor program mbed-os-example-clinet.bin
MDM: Chip is unsecured. Continuing.
MK64FN1M0VLL12.cpu: target state: halted
target halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x00000674 msp: 0x20030000
** Programming Started **
auto erase enabled
Flash Configuration Field written.
Reset or power off the device to make settings effective.
wrote 65536 bytes from file mbed-os-example-clinet.bin in 3.757346s (17.033 KiB/s)
** Programming Finished **
```
注意，这里的mbed-os-example-clinet.bin是同elf文件在编译时候一起生成的，你需要拷贝到跑openocd的目录下，否则会报错找不到文件。

接着，reset target，让硬件重新复位：
``` bash
(gdb) monitor reset init
```

接下来就可以用GDB的其他调试命令来调试了，这里就不多说。

## 测试 ##


# 内核分析 #

mbedOS很有趣，从编写一个应用程序开始你会发现它和其他的RTOS有些不同。为啥它的main()函数里不需要做任何操作系统初始化，甚至板级初始化，亦或是建立其他进程？mbdOS的内核从CMSIS-RTOS衍生而来，为客户提供了最大程度上的Out of Box Experience的使用体验。它把操作系统、硬件初始化的一切都在main()之前做完了，而且main()其实就是一个单独的进程，很有趣吧！



# 总结 #
