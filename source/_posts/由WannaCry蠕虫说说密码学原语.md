---
title: 由WannaCry蠕虫说说密码学原语
date: 2017-05-30
tags:
 - 密码学原语
 - HotSpot
category: 技术
---

WannaCry这阵子让很多人欲哭无泪， 感染蠕虫后，重要文件被AES算法加密， 而AES的key用RSA算法加密了. 网上各种分析和破解办法，然而没什么用，不能彻底的解决问题. 因为WannaCry的RSA+AES加密套餐，理论上，目前很难破解.  
密码学是很大的主题，简单扯扯密码学原语(cryptographic primitives).

<!--more-->

### 1. 简单介绍 cryptographic primitives

#### 1.1 Symmetric encryption (Secret Key)
对称加密提供Confidentiality.加密方和解密方用同一个key对数据进行加解密.
对大量数据加密时，因为非常对称加密算法的性能慢，我们一般用对称加密算法.
常见算法有：
* DES(Data Encryption Standard), 
* 3DES(triple DES)是个变体，在DES基础上来三次加密. 
* RC4(Ron Rivest in 1987),
* AES(Advanced Encryption Standard), 
AES是NIST(US Government's National Institute of Standards)2000提出的标准，目前的工业标准.

#### 1.2 Asymmetric encryption (Public Key)
非对称加密一般用作Authentication / Key exchange. 对称加密有个问题是加解密时用同一个key，那么key在发送方和接收方交换时要保证安全，可以用非对称加密算法来加密这个key.比如HTTPS协议，对数据全部用非对称加密性能上不能接受的，所以HTTPS协议在传输的数据用对称加密，客户端和服务器端交换密码时用RSA算法来加密对称加密的密码.  
常见有:
* RSA (图灵奖加持, 大名鼎鼎),  
* Diffie-Hellman, 
* ECC, ...

#### 1.3 Hash algorithms  => Integrity
下载文件时，常常会看到一个md5码，就是对文件hash值，主要时用来检查文件是否完整，没有被篡改.
常见的算法有
* MD5 , 
* SHA-1, SHA-2,  
* GMAC(a  variant of the GCM) …


#### 1.4 Authenticated Encryption(AE):  Confidentiality  +  Integrity
AE是一种重要的primitive，提供加密的同时又有完整性检查. 常见的有：
* RC4 + HmacMD5,  
* RC4 + HMAC-SHA-1, 
* AES-GCM, …


#### 1.5 Random Number Generator
* PRNG(Pseudo-Random Number Generator) 
* TRNG(True Random Number Generator)

Java的Random是个PRNG，给定seed，生成的序列其实是固定的.在生成随机密码时，伪随机的安全强度显然要弱很多.
而SecureRandom是安全的.

#### 1.6 Mode of operation
这个概念提一下，用对称加密算法对一个数据块(block, 16bytes)加密时，算法是固定的，相同输入和key，输出是一致的.为了增加安全性，为了混淆加密输出结果，引入各种办法, 就是所谓的CTR, CBC, GCM 等等modes. CTR支持随机读.


### 2. 怎样选择加密算法
选择加密算法时，考虑安全性和性能，还有依赖的系统的支持程度.
当然了，如果像csdn那样明码保存用户名和密码，那么可以避免选择加密算法的麻烦 :-)

#### 2.1 安全性
对称加密安全性：
* DES 被废弃，不应该选择使用
* 3DES，强度不够，灰常的慢，虽然目前使用的很多，不建议用.
* RC4 以快而出名，使用很多，但Mozilla和Microsoft都建议不用，RFC 7465禁止使用RC4
就安全强度而言，对称加密算法最好选择AES.

references:  
"Mozilla Security Server Side TLS Recommended Configurations". Mozilla. Retrieved 2015-01-03.  
"Security Advisory 2868725: Recommendation to disable RC4". Microsoft. 12 November 2013   

hash算法安全性：  
MD5早被王小云教授给破解， 前不久还听说过用集群破解SHA-1的新闻，随着硬件CPU/GPU/FPGA/ASIC的发展，SHA会越来越不安全.GMAC强度不错，是比较安全的算法.

#### 2.2 性能
只看对称加密算法的性能对比：
| algo               | throughput   |
| ------------------ | ------------ |
| 3des               |  14.59 MB/s  |
| aes-ctr-128(JDK8)  |  228.98 MB/s |
| aes-ctr-128(JDK8)  |  228.98 MB/s |
| aes-ctr-128(JDK8)  |  228.98 MB/s |


可见DES/3DES除了加密强度不够，性能也很差. rc4虽然速度还不多，但是安全性不推荐.
JDK8与Commons Crypto的差距在于工程上的实现问题.

生产随机密码，建议用TRNG,
| algo                 | throughput   |
| -------------------- | ------ ------|
| SecureRandom(JDK8)   |  14.59 MB/s  |
| CryptoRandom(Crypto) |  86.83 MB/s  |



### 3. 探究 AES 的性能秘密

除了硬件加速，比如x86平台有AESENC指令，Power/ARM上也有硬件加速指令；算法工程实现也会导致很大的性能差距.  
有空了再写写x86的代码优化，涉及到CPU的cache, pipelining, out of order, SIMD(SSE,AVX)等细节.


### 4. 其他

#### 4.1 细看AE算法AES-GCM
HTTPS, SSL/TSL的基础是AE，前面也说了RC4, MD5, SHA等算法安全强度不够.
Hanno Böck说:
> "If CBC/HMAC and RC4 are bad there's only one cipher mode left: AES-GCM (Galois/Counter Mode). "


GCM作为安全协议的底层算法可能会应用的更广.

我看过AES-GCM在OpenSSL, go runtime, Linux kernel的实现, 优化的很好，都是基于[Shay Gueron博士的算法](https://crypto.stanford.edu/RealWorldCrypto/slides/gueron.pdf).
看下benchmark：

| algo                 | throughput   |
| -------------------- | ------------ |
| RC4 + HmacMD5(JDK8)  |  149.8 MB/s  |
| RC4 + HmacSHA1(JDK8) |  131.69 MB/s |
| AES-GCM(JDK8)        |  3.98 MB/s   |
| AES-GCM(JDK9)        |  267.78 MB/s |
| AES-GCM(Crypto)      |  1196.01 MB/s|



可见：
* AES-GCM性能不比传统的RC4+Hmac差
* AES-GCM在JDK8里性能是个灾难
* AES-GCM在JDK9里用上了硬件加速，PCLMULQDQ指令

JDK9的实现还是不足够好，我本来想优化这个到HotSpot里去的，做了一半，被砍掉了，忙其他事情去了.

注意：   
[NONCE REUSE ISSUES IN TLS](https://int21.de/slides/berlinsec-gcm/#/)  
对GCM也有[不同的观点](https://int21.de/slides/berlinsec-gcm/#/11)

#### 4.2 吐槽 Java SASL API 
[Java SASL(Simple Authentication and Security Layer) API](https://docs.oracle.com/javase/7/docs/technotes/guides/security/sasl/sasl-refguide.html), 
内部实现时，一般是用MD5和3des/rc4来做数据完整性检查和加密的.从安全和性能角度，这算法的选择都不好.而SASL在大数据/分布式领域用的很多...

### 5. 小广告
#### 5.1 Apache Commons Crypto
因为JDK 7/8 不给力，搬Java砖的同志可以考虑用Apache Commons Crypto，工作所在team贡献的项目，又快又好，谁用谁知道:-)

#### 5.2 JDK9的改进
HotSpot上AES的实现有改进的地方，提了两patch，由公司的JVM team同事提交进去了   
JDK-8143925 加了个HotSpot Intrinsic, AES-CTR算法有5~8x的性能提升    
JDK-8152354 代码上微调，对CPU更友好，AES-CBC得到15%~50%的提升.  
Java 9今年7月应该可以GA了.
