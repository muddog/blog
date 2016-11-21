---
title: 透过mbedTLS了解DTLS握手协议
date: {}
tags:
- DTLS
- mbedTLS
- handshake
published: true
---
# (Understand the DTLS handshake by look into mbedTLS) #

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

## DTLS握手协议 ##

DTLS握手协议和TLS类似。DTLS协议在UDP之上实现了客户机与服务器双方的握手连接，并且在握手过程中通过使用RSA或者DH（Diffie-Hellman）实现会话密钥的建立。它利用cookie验证机制和证书实现了通信双方的身份认证，并且用在报文段头部加上序号，缓存乱序到达的报文段和重传机制实现了可靠传送。在握手完成后，通信双方就可以利用握手阶段协商好的会话密钥来对应用数据进行加解密。

放上我觉得很直观的两种握手（RSA/DH）过程的示意图帮助理解（TLS的，非DTLS）：
![](https://blog.cloudflare.com/content/images/2014/Sep/ssl_handshake_rsa.jpg)
![](https://blog.cloudflare.com/content/images/2014/Sep/ssl_handshake_diffie_hellman.jpg)

以及我画的简易握手流程图：
![](\DTLS-Analysis\dtls-handshake.png)

我们这里只讨论DH的握手协议，一般来说DH由于不需要传输Premaster Key，会比RSA的方式更安全。另外mbedTLS里的DTLS测试程序也使用了DH的握手协议。

## 通过mbedTLS来分析 ##

mbedTLS（前身PolarSSL）是面向嵌入式系统，实现了的一套易用的加解密算法和SSL/TLS库，并且mbedTLS系统开销极小，对于系统资源要求不高。mbedTLS是使用Apache 2.0许可证的开源项目，使得用户既可以讲mbedTLS使用在开源项目中，也可以应用于商业项目。使用mbedTLS的项目很多，例如Monkey HTTP Daemon，LinkSYS路由器。

我们在这里简单的利用mbedTLS自带的dtls_client/dtls_server的测试程序来分析握手协议：
1.在mbedTLS编译阶段，提升DEBUG LEVEL。利用mbedTLS的调试信息来理解程序流程
2.在Linux上跑dtls测试程序（localhost loopback），并且抓包

首先把mbedTLS下载下来：
``` bash
$git clone https://github.com/ARMmbed/mbedtls
```

然后在program/dtls_client.c, dtls_server.c里，调下DEBUG_LEVEL，然后将my_debug里加上时间戳信息，方便对比客户端和服务端的流程：
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

最后在Linux下直接make,DTLS的测试客户和服务端都会在program/下编译出来。然后打开wireshark之类的抓包工具，对本地网卡进行抓包。最后先跑server，后跑client。在抓包结束后，加dtls的display filter既可以：

_Wireshark抓包结果：_
![](\DTLS-Analysis\dtls-flow-capture.png)



## 参考文献 ##
- [Keyless SSL: The Nitty Gritty Technical Details](https://blog.cloudflare.com/keyless-ssl-the-nitty-gritty-technical-details/)
- [DTLS协议中client/server的认证过程和密钥协商过程](https://segmentfault.com/a/1190000006233845)
- [X.509 wiki](https://en.wikipedia.org/wiki/X.509)
- [mbedTLS Web](https://tls.mbed.org/)
- [RF4347](https://tools.ietf.org/html/rfc4347#section-4)
