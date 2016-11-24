---
title: 借助mbedTLS了解DTLS握手协议
date: {}
tags:
  - DTLS
  - mbedTLS
  - handshake
published: true
---
# (Understand the DTLS handshake by mbedTLS) #

## DTLS简介 ##

简单说，DTLS（Datagram Transport Layer Security）实现了在UDP协议之上的TLS安全层。由于基于TCP的SSL/TLS没有办法处理UDP报文的丢包及重排序（这些问题一般交给UDP的上层应用解决），DTLS在原本TLS的基础上做了一些小改动（大部分复用TLS的代码）来解决这些问题：
1. TLS记录层内记录的强关联性及无序号
2. 握手协议的可靠性
 - 包丢失重传机制（UDP无重传机制）
 - 无法按序接收（握手需要对包顺序处理）
 - 握手协议包长导致的UDP分包组包（类似于下层的UDP至于IP的分包）
3. 重复包（Replay）检测

由于UDP/DTLS相较于TCP/TLS的轻量化及较小的开销，目前被更多的运用的嵌入式环境中。例如CoAP使用DTLS来实现安全通路，CoAP及其上层的LWM2M则运用在物联网和云端的通讯上。

如果你已经很熟悉TLS，那么看到这里后就请忽略此文:)

## 如何借助mbedTLS来分析握手协议 ##

mbedTLS（前身PolarSSL）是面向嵌入式系统，实现了的一套易用的加解密算法和SSL/TLS库，并且mbedTLS系统开销极小，对于系统资源要求不高。mbedTLS是使用Apache 2.0许可证的开源项目，使得用户既可以讲mbedTLS使用在开源项目中，也可以应用于商业项目。使用mbedTLS的项目很多，例如Monkey HTTP Daemon，LinkSYS路由器。

我们在这里简单的利用mbedTLS自带的dtls_client/dtls_server的测试程序来分析握手协议。这里要说明的是mbedTLS这个自带的DTLS 测试程序，服务器只在localhost做bind，客户端也只连接localhost。实际上是loopback。并且使用了TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384的Cipher Suite，也就是用椭圆曲线（EC）的DH算法来实现密钥（Session Key）的协商，用RSA来实现密钥协商时交换的ECC类型，DH公钥等数据的签名加密。所以握手的流程和其他一些加密方式会有所差别。比如和单纯的RSA密钥交换方式比起来，会多一个"Server Key Exchange"报文。

### 下载mbedTLS ###

``` bash
$git clone https://github.com/ARMmbed/mbedtls
```

### 使能调试 ###

在program/dtls_client.c, dtls_server.c里，调下DEBUG_LEVEL，然后将my_debug里加上时间戳信息。使能协议栈的内部调试信息，方便对比客户端和服务端的流程：
``` C
#define DEBUG_LEVEL 100

static void my_debug( void *ctx, int level,
                      const char *file, int line,
                      const char *str )
{
    struct timeval tv;

    ((void) level);

    gettimeofday(&tv, NULL);
    //strftime(outstr, sizeof(outstr), "%H:%M:%S", tmp);

    mbedtls_fprintf( (FILE *) ctx, "[%06ld.%ld]%s:%04d: %s", tv.tv_sec, tv.tv_usec, file, line, str );
    fflush(  (FILE *) ctx  );
}
```

### 编译 ###

``` bash
$ make
```

DTLS的测试客户和服务端都会在program/下编译出来。

### 抓包 ###
打开wireshark之类的抓包工具，对本地网卡进行抓包。先跑server，后跑client。在抓包结束后，加dtls的display filter既可以：

_Wireshark抓包结果：_
![](\DTLS-Analysis\dtls-flow-capture.png)

有了TLS协议栈的调试信息，Wireshark的实际的抓包数据再加上源代码，我们就很容易来分析DTLS的握手协议。


## DTLS握手协议分析 ##

DTLS握手协议和TLS类似。DTLS协议在UDP之上实现了客户机与服务器双方的握手连接，并且在握手过程中通过使用RSA或者DH（Diffie-Hellman）实现会话密钥的建立。它利用cookie验证机制和证书实现了通信双方的身份认证，并且用在报文段头部加上序号，缓存乱序到达的报文段和重传机制实现了可靠传送。在握手完成后，通信双方就可以利用握手阶段协商好的会话密钥来对应用数据进行加解密。

简易握手流程图：
![](\DTLS-Analysis\dtls-handshake.png)

从流程图上看，有(1)(3)两个"Client Hello"请求，他两之间的区别是第二个"Client Hello"包含有(2)"Hello Verify Request"里服务端发来的Cookie。要使得DTLS握手正真开始，服务端必须要判断发送请求的是有效的，正常的客户端。通过这样的Cookie交互，可以很大程度上保护服务端不受DoS的攻击。如果不这么做，服务端会在收到每个客户请求后返回一个体积大很多的证书给被攻击者，超大量证书有可能造成被攻击者的瘫痪。当首次建立连接时，(1)请求包中的cookie为空，服务端根据客户端的源IP地址通过哈希方法随机生成一个cookie，并填入(2)"Hello Verify Request"包中发送给客户端。客户端收到Cookie后，再次发送带有该Cookie的"Client Hello"包(3)，服务端收到该包后便检验报文段里面的cookie值和之前发给该客户端的Cookie值是否完全相同，若是，则通过Cookie验证，继续进行握手连接；若不是，则拒绝建立连接。所以说(1)(2)步骤只在第一次连接时发生，之后在Cookie有效的情况下，DTLS握手从步骤(3)开始。

- 客户端的实现都在ssl_cli.c里，状态机由**mbedtls_ssl_handshake_client_step()**处理
- 服务端的实现则在ssl_srv.c里，状态机由**mbedtls_ssl_handshake_server_step()**处理

(3)"Client Hello"报文内容主要包含：
1. Random 32字节随机数，前4字节为当前时间+28字节随机数
3. Cookie，从报文(2)中获得
4. Cipher Suite，客户端可以支持的密钥交换方式
5. Compression methods，是否压缩，及压缩方式
6. Extension，例如服务器主机名，支持的签名加密方式，EC曲线的类型等

由函数**ssl_write_client_hello()**实现报文填充和发送。

服务端收到报文(3)后，会调用函数**ssl_parse_client_hello()**做一系列协商工作：
将Random保存；验证Cookie是否和客户端的IP匹配；根据客户端提供的Cipher Suite找最佳匹配的能提供的Cipher算法集合（包括Session Key交换方式和加密方式）。mbedTLS的例子里使用了ECDHE_RSA_WITH_AES_256_GCM_SHA384。如果协商没有问题，服务端就调用**ssl_write_server_hello()**会发送报文(4)"Server Hello"，告诉客户端使用什么Cipher Suite做握手，什么压缩方式，并且会利用随机数生成一个Session ID。并且以客户端同样的方式生成的Random随机数，将随机数和Session ID放入报文中。
紧接着服务端会接连发送报文(5)(6)(7)，将证书、生成Session Key的方法和参数发送给客户端。并用(7)"Server Hello Done"，告诉客户端Hello阶段结束。(5)"Certification"比较简单，将Server。。。
(6)"Server Key Exchange"






## 参考文献 ##
- [Keyless SSL: The Nitty Gritty Technical Details](https://blog.cloudflare.com/keyless-ssl-the-nitty-gritty-technical-details/)
- [DTLS协议中client/server的认证过程和密钥协商过程](https://segmentfault.com/a/1190000006233845)
- [大型网站的HTTPS实践（一）---HTTPS协议和原理](http://blog.csdn.net/luocn99/article/details/45460673)
- [X.509 wiki](https://en.wikipedia.org/wiki/X.509)
- [mbedTLS Web](https://tls.mbed.org/)
- [RF4347](https://tools.ietf.org/html/rfc4347#section-4)
- [RF4492](https://tools.ietf.org/html/rfc4492#section-2)
