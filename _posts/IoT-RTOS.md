---
title: IoT的那些操作系统
date: {}
tags:
- IoT
- RTOS
- Embeded
published: true
---

# 引子 #
IoT(Internet of Things)一直是个市场没爆发，但炒作够火的概念。 最近看了Brillo（Goolge放出的OS for IoT），再结合手头做的项目，来聊聊打着IoT旗号的那些操作系统及其生态。可以让大家在对此类嵌入式系统软件平台选型时少些困惑。

首先，不在这里描述IoT到底是啥，这概念太笼统。但凡事都讲套路，在我看来大部分的IoT操作系统都是顺着这个套路（应用场景）来：


        Internet            PAN
	Cloud <-> Board router <-> Nodes (things connected)
       ^          ^             ^
       |          |             |
        ---- Smart device ------
          (e.g. phone, tablet)


这场景基本适用于家庭、楼宇及工业等等。所以哪家提啥方案都可以按这个套路来，下面你会看的到。文中我们会谈到以下这些OS：

- Brillo (a solution from Google for building connected devices)
- mbedOS (ARM mbed, The ARM IoT Device Platform)
- RIOT (The friendly Operating System for the IoT)
- Contiki (an open source operating system for the IoT)
- Zephyr (a small, scalable real-time operating system for use on resource-constrained systems)
- Nuttx (a RTOS with an emphasis on standards compliance and small footprint)

除了单独对这些OS介绍外，我们也会来个横向大比拼。目的是让项目负责人，产品经理在对系统选型的时候能有个合理的参考。嵌入式系统中，最好用，生态最好的当然是Linux。但当系统较小，能力不足以跑Linux的时候，就很容易纠结，特别是对于生态要求很高的应用。

# Brillo #
[https://developers.google.com/brillo/](https://developers.google.com/brillo/)

> Brillo: Google’s OS for IoT MPU devices
> • Targeted at smart homes
> • Expanding to buildings and industry
> • Supports MPU devices w/ min 35MB of RAM

Brillo需要跑在带MMU的AP上。其实很显然，Brillo基于Android，它再怎么裁剪，也是需要跑在Linux，Kernel上还要打一堆patch。只是它把Android上关于图形、JAVA虚拟机及Framework统统裁减掉。只保留了C/C++运行环境，Binder IPC，SSL等网络安全必须组件。这也就意味着在Brillo上开发APP其实是Native的，而且驱动程序都由Android的那套HAL来做抽象，所以应用程序是直接和HAL、Lib来打交道。

<!-- more -->

*Brillo 架构框图*
![brillo architecture](IoT-RTOS/brillo_arch.jpg)

看似Google放了个IoT的操作系统出来，实际Google目前专注于应用场景中功能相对强大的节点，或者边界节点Border Router这样的角色。最重要的是Google利用Weave打通了设备节点到Google云端的通道。Google实际上是用了最小的代价，实现了移动设备平台Android、物联网节点和自己的云端的互联互通。至于那些跑不了Brillo的节点怎么互联互通，Google交给其他人去考虑了，反正Brillo也支持6LowPan, Thread之类的协议。如果你写过Android HAL/Service，那么开发Brillo App易如反掌，直接调用Device HAL去操作设备；调用Weave API去做通讯。

``` C
    int ret = hw_get_module(LIGHTS_HARDWARE_MODULE_ID, &module);
    if (ret || !module)
        err(1, "Failed to load %s", LIGHTS_HARDWARE_MODULE_ID);
    ret = module->methods->open(module,    
                                LIGHT_ID_NOTIFICATIONS,
                                reinterpret_cast<struct hw_device_t**>(&light_device));
    if (ret || !light_device)
        err(1, "Failed to open %s", LIGHT_ID_NOTIFICATIONS);
```

以上代码调用HAL统一接口打开一个LIGHT设备，是不是很熟悉？然后起个Daemon,利用Weave API来接收从网络上来的XMPP请求，对设备进行配置或者状态监测。在云端和移动端，可以使用网页版的Weave Developers Console和Weave App来控制监测设备。
Brillo可以在资源较少的MPU上跑，35MB内存就行，老的ARM9估计也可以。它可建立很多低成本的设备节点，用来和智能设备、云端通讯，或者作为PAN内设备对外的桥梁Border router。Brillo可玩性应该很高，家里的PC机，路由器都可以拿来跑，生态系统强大。但致命缺点是，我们在天朝啊，你懂得。相信国内的BAT之类，会考虑移植，改造，变异。就如对Android一般。

# mbedOS #
[https://www.mbed.com/en](https://www.mbed.com/en)

mbed是ARM自己建立的IoT解决方案平台

> The ARM mbed IoT Device Platform provides the operating system, cloud services, tools and developer ecosystem to make the creation and deployment of commercial, standards-based IoT solutions possible at scale.

它被ARM分成三大部分

1. **mbed Cloud**
2. **mbed Device Connector**
3. **mbed Client**

ARM居然也自己搞了个Cloud，可以通过“mbed Device Connector”来访问连接到云端的设备。并提供网页版的Connector来管理设备，用户可以通过RESTful API over HTTP来写自己的APP。

“mbed Client”的定义是这样的：
> a library that connects devices to mbed Device Connector Service

看似比较奇怪，其实就是一套可以移植到各种操作系统上的，能够和mbed Device Connector Service通讯的，跑在硬件设备上的软件库。它使用基于UDP的CoAP协议来通讯，使用mbedTLS来实现安全连接，兼容LWM2M。

*mbed Client 架构框图*
![mbed_client](IoT-RTOS/mbed_client.jpg)

说到这里，我们的主角mbedOS该登场了。ARM为了在基于ARM Cortex-M内核的硬件平台上实现对设备的操作，及通过Device Connector访问云端，它必须有一套可以支持mbed Client的软件解决方案。mbedOS既是基于RTOS内核，并提供各种ARM SoC硬件平台驱动和BSP的操作系统，在此之上，实现整个mbed Client库。在PAN的物联区域内，设备与设备的通讯都可以使用mbedOS提供的方案来解决，它支持NFC，RFID，BLE，6LowPAN甚至是Thread。在设备与云端的通讯上，mbedOS既支持以太网，WiFi，也支持3G。跑mbedOS的设备既可以是设备节点，也可以是边界路由器（比如6LowPAN转IPv6），也可以和智能设备通过BLE通讯。mbed Device Connector又可以使用CoAP协议在设备端和云端通讯。

mbedOS对于网络的支持可谓很强大
- LWIP IPv4/v6, TCP/UDP
- mbed BLE stack
- 6LowPAN (host, router, border router)
- Thread (ED, router, border router)
- BSD socket API

除此之外，它还支持
- 文件系统：cfstore, flash-journal等
- C++的驱动接口，及驱动抽象层HAL
- 几乎所有使用ARM CortexM核的大厂硬件平台。广度可以，但深度有待提高

也就是说mbed把每一类驱动都抽象成一个基类，真正做驱动移植的时候，从这个类派生出来，然后实现相应的HAL函数。使得用户在实例化该驱动派生类后，能够调用相应的类接口，从而访问实际的设备驱动程序。比如AnalogIn::read()，是返回ADC的采样结果。

再看mbed生态
- ARM提供在线IDE，可以在线快速编译
- 线下的开发环境也简单易用, mbed cli类似于android的repo
- github上的例子很多，参考性强
- 社区相对比较活跃
- 合作伙伴众多

ARM搞起了云和RTOS，令人联想到之前的Linaro，看似前景不错。而且据说国内的BAT也有在使用mbed做产品，其实就把mbed Cloud换自己的云，改造下Device Connector即可。本人也正在研究mbed，之后会写些使用心得。

# RIOT #
[https://riot-os.org/](https://riot-os.org/)

RIOT官方的口号:)

> if your tiny IoT device can’t run Linux, use RIOT

RIOT是面向开发者的，开源的，适合物联网的操作系统。它的背后没有某个公司的支持，而完全是由社区驱动。
他的一些特性：
- 标准的C/C++编程
- 标准的gcc编译环境
- 可以跑在8位，16位和32位的嵌入式系统上
- 部分的POSIX接口兼容（以后的目标是全兼容）
- 支持在Linux/Unix的虚拟机上运行
- 实时性，快速的中断响应（~50 clock cycles）
- 微内核，组件都可以动态加载，并且通过message来实现服务
- 极小开销的多线程支持（< 25 bytes per thread）
- 丰富的网络支持：6LoWPAN，IPv6，RPL，CoAP and CBOR
- 高精度的定时器
- 丰富的工具 （System shell, SHA-256, Bloom filters, ...）

*RIOT 架构框图*
![riot architecture](IoT-RTOS/riot_arch.jpg)

RIOT的CPU的IP驱动基本都有一套统一接口，但是没有任何抽象层，被放在源代码的cpu\<cpu>\periph中。这意味着在做新的平台支持时，你要注意驱动的接口要和API文档里的一致，比如ADC的adc_init(), adc_read()。源代码的drivers\则放着板级的驱动，比如NXP的MMA8541，利用i2c统一接口来访问。
由于是微内核（microkernel）的实现，所有的系统服务包括时钟、网络协议栈、网络服务等，都是通过创建独立的线程来实现。在线程中都有event_loop来接收服务请求，处理并发送服务结果。RIOT中最关键的是GNRC（Generic network stack）网络协议栈，它实现了从MAC层一直到传输层的各种协议，如6LowPan，IPv4/v6，RPL，TCP/UDP。并且这些不同的协议栈之间通过netapi统一接口开放给用户。对于应用层来说，GNRC提供了conn和socket两种API。在安全方面，貌似802.15.4这层没有加入AES的支持，只提供tinyDTLS在应用层给用户使用。由于RIOT的POSIX的部分兼容性，及提供BSD socket的接口，很多应用都可以方便的移植过来，在pkg/你能找到例如libcoap，openwsn这样的应用。
RIOT最早是由柏林自由大学开发的，目前完全由社区维护，貌似欧洲开发者居多。从devel maillist里来看，感觉社区活跃程度一般。每两周都有一个Virtual meeting，都还是大学在牵头。
总之，一个很有想法的微内核，加上开发环境相对于之前熟悉Linux的开发者来讲很友好。应该是个潜力股。

# Contiki #
[http://www.contiki-os.org/](http://www.contiki-os.org/)

以下是维基百科对Contiki的介绍：

> Contiki is an operating system for networked, memory-constrained systems with a focus on low-power wireless Internet of Things devices. Extant uses for Contiki include systems for street lighting, sound monitoring for smart cities, radiation monitoring, and alarms

可以看得出来，原来Contiki是为智能城市而诞生的。支持的平台有限，基本是内部集成CC24xx/25xx，MC1322x之类Radio的SensorTag平台，或者一个很小的MCU加上这些Radio模块的平台。从它所支持的平台也能看出，Contiki更加专注于小型传感器节点。它更关注与PAN内的节点通讯，当然他也有传统的IPv4/v6，TCP/UDP支持，使得利用CoAP可以用来和云端通讯。

Contiki的特性：
- 完整的网络支持，HTTP，UDP/TCP，以及低功耗协议6lowpan，RPL和CoAP。整个IPv6协议栈都是有思科贡献
- 专门为小内存设备设计的内存管理器
- 小巧的估算功耗的工具
- 丰富的实例
- 高效的支持外部Flash的Coffee flash文件系统
- Protothreads，事件驱动及多线程的编程模型
- Cooja网络模拟器
- Rime协议栈，比IPv6更轻量级的网络层

Contiki也是个微内核（microkernel），所有的系统服务都是通过启线程完成，Protothreads线程整合了线程间事件通讯，使得编写系统服务非常容易。驱动程序方面，Contiki没有统一的驱动程序框架，驱动都是各家MCU自带开发包提供，这样的好处是能够保证生成的二进制代码够小。
Contiki有个很有特色的模拟器，Cooja Network Simulator，可以运行很多例子，并且可以监控整个网络的包及节点状态。这样可以让用户在没有充足硬件设备的条件下做开发。
Contiki社区基本依靠maillist讨论问题，github做pull request。从maillist archive里看，社区活跃程度很一般。

# Zephyr #
[http://www.zephyrproject.org/](http://www.zephyrproject.org/)

Zephyr居然是Linux基金会的合作项目。应该是由INTEL将WindRiver的商用操作系统WindRiver Rocket部分开源后诞生的项目（今年才诞生）。目前可用资料不多，而且支持的硬件平台较少，ARM的平台就没几个。
Zephyr像极了Linux，它的源代码目录结构Kconfig使用方式，启动流程，Driver Model都可以看出来。用户的应用程序是启动后创建的最后一个线程，用户的main()会在所有的驱动，组件及硬件板子初始化后被调用，驱动使用DEVICE_INIT()，组件使用SYS_INIT()初始化，并带有优先等级。驱动都会遵循统一的驱动结构：
``` C
struct device {
        struct device_config *config;
        const void *driver_api;
        void *driver_data;
};
```
所以驱动要自己定义一个配置结构，一套API的函数指针，以及驱动状态结构。和Linux很像。
网络支持很奇葩，协议栈居然都是用的Contiki的，Bluetooth还算比较全。子系统方面，文件系统、USB都是很简单，很原始的支持。安全方面，mbedTLS和tinyDTLS被拿了过来。
总体来讲，Zephyr还处于初期，很多东西都不完善。Owner又是Intel，希望别重蹈Moblin的覆辙。


# Nuttx #
[http://www.nuttx.org/](http://www.nuttx.org/)

Nuttx,实时操作系统，POSIX接口支持，Loadable内核模块支持，BSD socket，MMU支持，等等。我只能说，长的太像Linux了。Build也是Kconfig，目录结构也基本和Linux Kernel一样。

- ARM的核基本都支持
- 文件系统也是VFS支撑，大而全。网络的，NAND MTD的，pseudo都支持
- 自己的Clib,也可以支持uCLib
- 全面的网络协议栈，但是没有wireless！
- 有自己的USB协议栈

我一开始没打算谈Nuttx，他的无线支持很差，就更别谈无线互联了。但是BAT居然有人用它做云OS。估计是团队实在太熟悉Linux了，跑不了Linux也要找个类似的开发环境。有了BAT的支持，Nuttx估计可以发达了。

# 大比拼 #

这里用一张比较简单的表来对比这些操作系统和生态

|   OS   | Brillo | mbedOS | RIOT | Contiki | Zephyr | Nuttx |
| ------ | ------ | ------ | ---- | ------- | ------ | ----- |
| **Realtime** | No | Yes | Yes | Yes | Yes | Yes |
| **Footprint (RAM/ROM)** * | 35M RAM | 32K/256K | ~1.5K/~5K | < 10K/30K | 8K RAM | 16K/32K |
| **Dev Language** | C| C/C++| C/C++| C| C| C/C++| 
| **MCU w/o MMU** | No| Yes| Yes| Yes| Yes| Yes| 
| **CPU core**| Cortex-A| Cortex-M| Cortex-M, ARM7, x86| Cortex-M, x86| Cortex-M, x86| Cortex-A/M/R, x86, MIPS| 
| **Networking**| Weave, 802.15.4 (Zigbee, Thread), BLE, WiFi, Ethernet | CoAP, BLE, 6LowPAN, Thread, WiFi, Ethernet, Cellular | CoAP, BLE, 6LowPAN, RPL, Ethernet | CoAP, 6LowPAN, RPL, Ethernet | CoAP, BT4.0, BLE,  6LowPAN, Ethernet | Ethernet| 
| **Cloud Service** | Yes | Yes | No | No | No | No |
| **Build System**| repo/gcc| mbed/gcc (armcc)| gcc| gcc| kconfig/gcc| kconfig/gcc|
| **License**| Apache 2.0| Apache 2.0| LGPLv2| 3-clause BSD| Apache 2.0| BSD| 
| **Repository**| N/A| github| github| github| N/A| bitbucket| 

*对于Footprint来说，基本都是各自网站上的宣称值。除了Brillo以外，其他都是RTOS有很小的内核。程序所占的内存和代码大小取决你需要的硬件平台的驱动多少，需要什么样的协议栈等等的功能。*

对于我个人来讲，我更愿意去研究、使用Google, ARM这些牛X公司推动的产品。他们有钱、牛X，能建立庞大的生态圈，整合更多的资源。而且社区相对活跃，有啥问题也容易解决。

# 结尾 #
最近行业趋势有所变化，除了互联网依然火热外，大家的焦点无疑都从手机投向了汽车、工业、物联网。大家都希望能够使用物联网和互联网将传统行业中的产品实现互联互通，实现信息共享、提高生活、生产效率和质量。我们所工作在的半导体行业中，除了给市场提供更符合需求的处理器以外，我们还需要提供基于自己处理器的软硬件解决方案。操作系统作为应用的基础、基石，显得非常的重要。本文只是粗劣的介绍和对比了这些物联网的操作系统，希望能对读者有所帮助。任何意见建议都欢迎联系我。
