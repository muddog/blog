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

_Wireshark抓包 [下载](\DTLS-Analysis\dtls.pcapng) 截图：_
![](\DTLS-Analysis\dtls-flow-capture.png)

有了TLS协议栈的调试信息，Wireshark的实际的抓包数据再加上源代码，我们就很容易来分析DTLS的握手协议。


## DTLS握手协议分析 ##

DTLS握手协议和TLS类似。DTLS协议在UDP之上实现了客户机与服务器双方的握手连接，在握手过程中验证对方的身份，并且使用RSA或者DH（Diffie-Hellman）实现会话密钥的建立，以便在后面的数据传输中对数据加密。它利用cookie验证机制和证书实现了通信双方的身份认证，并且用在报文段头部加上序号，缓存乱序到达的报文段和重传机制实现了可靠传送。在握手完成后，通信双方就可以利用握手阶段协商好的会话密钥来对应用数据进行加解密。

简易握手流程图：
![](\DTLS-Analysis\dtls-handshake.png)

从流程图上看，有**(1)(3)**两个**"Client Hello"**请求，他两之间的区别是第二个包含有**(2)"Hello Verify Request"**里服务端发来的Cookie。要使得DTLS握手正真开始，服务端必须要判断发送请求的是有效的，正常的客户端。通过这样的Cookie交互，可以很大程度上保护服务端不受DoS的攻击。如果不这么做，服务端会在收到每个客户请求后返回一个体积大很多的证书给被攻击者，超大量证书有可能造成被攻击者的瘫痪。当首次建立连接时，**(1)**请求包中的cookie为空，服务端根据客户端的源IP地址通过哈希方法随机生成一个cookie，并填入**(2)"Hello Verify Request"**包中发送给客户端。客户端收到Cookie后，再次发送带有该Cookie的**"Client Hello"**包**(3)**，服务端收到该包后便检验报文段里面的cookie值和之前发给该客户端的Cookie值是否完全相同，若是，则通过Cookie验证，继续进行握手连接；若不是，则拒绝建立连接。所以说**(1)(2)**步骤只在第一次连接时发生，之后在Cookie有效的情况下，DTLS握手从步骤**(3)**开始。

- 客户端的实现都在ssl_cli.c里，状态机由**mbedtls_ssl_handshake_client_step()**处理
- 服务端的实现则在ssl_srv.c里，状态机由**mbedtls_ssl_handshake_server_step()**处理

**(3)"Client Hello"**由函数**ssl_write_client_hello()**实现报文填充和发送，内容主要包含：
1. Random 32字节随机数，前4字节为当前时间+28字节随机数
3. Cookie，从报文(2)中获得
4. Cipher Suite，客户端可以支持的密钥交换方式
5. Compression methods，是否压缩，及压缩方式
6. Extension，例如服务器主机名，支持的签名加密方式，EC曲线的类型等

服务端收到报文**(3)**后，会调用函数**ssl_parse_client_hello()**做一系列协商工作：
将Random保存；验证Cookie是否和客户端的IP匹配；根据客户端提供的Cipher Suite找最佳匹配的能提供的Cipher算法集合（包括Session Key交换方式和加密方式）。mbedTLS的例子里使用了ECDHE_RSA_WITH_AES_256_GCM_SHA384。如果协商没有问题，服务端就调用**ssl_write_server_hello()**会发送报文**(4)"Server Hello"**，告诉客户端使用什么Cipher Suite做握手，什么压缩方式，并且会利用随机数生成一个Session ID。并且以客户端同样的方式生成的Random随机数，将随机数和Session ID放入报文中。
紧接着服务端会接连发送报文**(5)(6)(7)**，将证书、生成Session Key的方法和参数发送给客户端。并用**(7)"Server Hello Done"**，告诉客户端Hello阶段结束。

**(5)"Certification"**服务端会将他自己的证书发送给客户端。证书中的肯定有一个证书的Subject是和Server Name相匹配的（commonName=localhost，organizationName="PolarSSL"）。测试程序中其实发送了3个证书，subject分别是localhost, PolarSSL Test CA及PolarSSL Test EC CA。PolarSSL Test CA实际上是localhost的父证书，这里就涉及到一个CA Chain的概念。现实情况中，服务器发给客户端的证书一般是终端用户证书（end-user CA），那么客户端怎么知道这个服务器是可信的？客户端必须在自己安装的一系列中间证书（intermediate CA）和根证书（ROOT CA）中查找终端用户证书的父证书，以及该父证书的父证书，直到根证书，建立起CA Chain。一般来讲，终端客户证书的Issuer都会和某个中间或者根证书的Subject相匹配，也就意味着客户证书是由某个中间或根证书发行机构签发，并且终端客户证书中的签名是由父证书拥有者的私钥加密的。所以客户端建立起CA Chain后，会首先使用父证书的公钥来验证终端用户证书的完整性，然后找到父证书的父证书，以同样的方式验证父证书完整性，直到遇到根证书。由于根证书的Subject和Issuer都是自己，所以客户端的根证书一定要保证是Trust CA颁发的，否则没有办法自己验证自己。
![CA Chain](https://upload.wikimedia.org/wikipedia/commons/thumb/d/d1/Chain_of_trust.svg/645px-Chain_of_trust.svg.png)
回到我们的例子，客户端在初始化的时候加载了PolarSSL Test CA的根证书（当然只是测试的根证书），这个和现实的情况类似。所以客户端收到**(5)"Certification"**报文后，很快就可以查找到"localhost"这个终端用户证书的根证书是"PolarSSL Test CA"，一下便验证通过了。验证的代码在：**mbedtls_x509_crt_verify_with_profile()**。验证通过后，客户端实际上就获得了证书中的公钥。

**(6)"Server Key Exchange"**是Session Key协商的重要一步**ssl_write_server_key_exchange()**，测试程序中使用了ECDHE_RSA的协商方式。服务端首先要将在**(3)"Client Hello"**阶段和客户端协商使用的EC曲线类型找出来，这里协商的曲线类型是secp512r1(0x0019)。利用该曲线类型，加载对应于该曲线的参数p,b,G(x,y),n （服务端和客户端参数相同，参考：library/ecp_curves.c）。调用**mbedtls_ecdh_gen_public()**，首先生成一个随机数私钥a（范围[1, n-1]），然后计算Qs=aG，产生ECDHE的Public Key Qs，再用服务端的私钥（和localhost证书中的公钥配对的）对Qs做签名（SHA512做哈希，RSA加密）。最后将曲线类型，公钥Qs，签名算法及签名写入**(6)**的报文中，发送给客户端。由于ECDHE(Elliptic curve Diffie–Hellman)算法较为复杂，我也不是非常理解，具体可以参考：
[WikiPage](https://en.wikipedia.org/wiki/Elliptic_curve_Diffie%E2%80%93Hellman)
我们在这里就不深入讨论算法了。总之，ECDHE算法很快，而且不需要暴露预主密码（premaster secret），后面客户端也会做同样的操作，生成自己的私钥b,Qc。双方交换Q后，可以计算的到相同的预主密码:bQs=baG=abG=aQc。之后双方就可以用这个abG预主密码和之前(3)(4)报文中的客户端、服务端的Random来生成对称密钥Session Key（master secret）。预主密码如何生成密钥，可以参考**mbedtls_ssl_derive_keys()**：
```
master = PRF( premaster, "master secret", randbytes )[0..47]
```
因为premaster secret不需要做交换，而是在本地计算产生，所以说ECDHE的密钥协商方式比RSA更安全。

**(7)"Server Hello Done"**会紧接着(6)发送，内容很简单，标示了handshake type 14,，告诉客户端Hello阶段结束。

**(8)"Client Key Exchange"**客户端在收到(6)报文后，获取服务端发送过来的EC曲线类型和Qs，使用和服务端一样的流程生成私钥b，计算公钥Qc=bG，将Qc放入该报文，并发送给服务端。此时不需要再做签名。

**(9)"Change Cipher Spec"**客户端接着发送该报文，告诉服务端Session Key我已经生成，我们可以用密钥开始加密通讯了。

**(10)"Finished"**该报文由客户端发送**mbedtls_ssl_write_finished()**，这是第一个用Session Key加密的密文，内容为：
```
valid_data = PRF(master_secret, "client finished", MD5(handshake_messages) + SHA(handshake_messages))[0..11]
```
handshake_messages实际上是客户端在握手阶段发出的所有报文（除开该Finished报文），服务端在收到该报文后，会以同样的方式计算出valid_data，并且做比较。以确认双方协商的密钥一致。确认完毕后，服务端以同样的方式发送**(11)"Change Cipher Spec"**和**(12)"Finished"**给客户端。最后完成握手。


## 参考文献 ##

- [DTLS协议中client/server的认证过程和密钥协商过程](https://segmentfault.com/a/1190000006233845)
- [大型网站的HTTPS实践（一）---HTTPS协议和原理](http://blog.csdn.net/luocn99/article/details/45460673)
- [X.509 wiki](https://en.wikipedia.org/wiki/X.509)
- [mbedTLS Web](https://tls.mbed.org/)
- [RF4347](https://tools.ietf.org/html/rfc4347#section-4)
- [RF4492](https://tools.ietf.org/html/rfc4492#section-2)
