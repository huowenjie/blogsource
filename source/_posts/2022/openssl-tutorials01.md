---
title: Openssl 加密算法库编程精要 01 简介
date: 2022-01-26 18:07:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

提示：如果读者发现了本系列文章所述的问题和错误，笔者欢迎各位读者将建议发到笔者的电子邮箱 huowenjie0427@sina.com，我将不定时更正或优化。

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. [Openssl Wiki](https://wiki.openssl.org/index.php/Main_Page)

## 1.1 简述

Openssl 是一个免费开源的工具包，实现了 TLS/SSL 协议和绝大多数的主流密码算法和标准，最初的版本由 Eric A. Young 和 Tim J. Hudson 创建，其官方网站为 [www.openssl.org](www.openssl.org)。Openssl 主要包含三个部分：

- libssl  -- TLS 协议的实现（RFC 8446）
- libcrypto -- 加密算法库，支持绝大多数密码算法，是实现 TLS 协议和 PKI 体系的基础
- openssl -- 命令行工具

Openssl 当前主要的发布版本是 1.1.1，新开发的 Openssl3.0+ 相较于前者不论是架构体系还是设计理念都有了较大的变化。尽管 Openssl 团队发布声明会尽量将这次的版本升级的影响降至最低，但是不可避免的，用户在从 1.1.1 迁移至 3.0+ 仍然需要改动部份程序，并且需要重新编译应用程序。所以目前来看 Openssl3.0 这个版本并不兼容旧的版本，而且随着时间的推移会逐步将一大部分接口直接废弃，特别是那些贴近底层的 API。
Openssl 团队进一步优化了新版本架构，将各个模块的耦合性进一步降低，并且提供了一个叫 “Provider” 的模式用来管理其他的底层模块，实现动态的、可插拔的底层实现。而对于应用层而言，用户无需考虑各种底层实现，只需要专注于功能和应用，以下是 Openssl3.0 的整体架构设计图（所有的设计图片均摘录自 Openssl3.0 官方设计草案，见参考资料）：

![](/image/2022/openssl-tutorials/OpensslDesign.png)

按照我的理解，“Provider” 是一个抽象的概念，更像是一个标准。每个不同的 “Provider” 按照相同的标准实现相应的底层算法和功能，然后通过 “Core” 模块直接 “装配” 到 Openssl 上，通过这种方式给顶层的应用程序提供密码服务。这样的设计增加了整体结构的灵活性，除了可以隔离应用层和底层，使底层的变化不至于大范围的影响应用层，还可以自己定制个性化的 “Provider” 以满足特殊的需求。
Openssl 3.0 的内核设计也是非常的有特色，内核可以缓存和调度各个 Provider, 使其可以各自独立地工作互不影响，并且它和 EVP API 模块有着非常紧密的联系，如图所示：

![](/image/2022/openssl-tutorials/OpensslArch.png)

用户首先会通过内核加载 Provider，在加载 Provider 的时候内核会将该 Provider 的相关信息包括算法实现全部加载到缓存区域，用户在使用该 Provider 提供的算法时会首先 “Fetch” 算法（其实就是查询），然后内核会将该算法的实现提供给 EVP，然后用户通过 EVP 模块调用算法，EVP 在这个过程扮演了一个代理的角色。

这里只是简单介绍一下 Openssl3.0 的设计理念，其他具体的设计方案和代码示例，读者均可在 openssl 官网上查到，参考网址可见本节末尾，这里不再讨论。

我在这里特别说明一下：由于 Openssl 的代码量十分庞大，单一个加密算法库就有数十万行的代码，而且由于密码学博大精深，相关的数学原理更是晦涩难懂，笔者能力有限，不可能精通所有相关的知识，所以本系列的文章并不会针对 Openssl 的源码进行深入的讨论，我只会在用户层面对 Openssl 的加密算法库做一些常用的 API 解读，重在应用！如果涉及到我相对了解的知识，我会尽量无保留地分享，文章有错误的话在所难免，我也欢迎各位读者对我文中的错误进行批评指正！共同学习，共同进步！

作者假定各位读者已经熟练掌握 C 语言和有基本的密码学知识。

## 1.2 编译和安装

Openssl 的编译十分简单，步骤如下：

（1）在官网下载 openssl3.0.1 版本，得到 openssl-3.0.1.tar.gz;  
（2）输入 tar -zxvf openssl-3.0.1.tar.gz 解压 tar 包到当前目录；  
（3）进入解压后的目录执行 ./Configure 生成 Makefile 文件（openssl 默认安装到 /usr/local 下，如需指定安装路径，则执行 ./Configure --prefix=/xxx/xxx）；  
（4）执行 make 编译；  
（5）执行 make install 安装。

## 1.3 常用的编译选项和配置

默认编译生成 release 版本的共享库，如果要生成可调式的库，则输入 ./Configure --debug，具体的编译选项请查询 INSTALL.md 文件。

## 1.4 Openssl3.0 加密算法库目录结构

虽然我们的目标并不是完全精通 Openssl 的源码，但是了解其源代码目录的含义仍然是有必要的：

- aes: AES 对称算法实现
- aria: ARIA 对称算法实现
- asn1: ASN.1 编解码实现
- aysnc: 异步线程池实现
- bf: blowfish 对称算法实现
- bio: I/O 流的抽象
- bn: 大数运算实现
- buffer: 内存缓冲区
- camellia: Camellia 块密码算法
- cast: CAST 对称算法
- chacha: ChaCha20 流密码算法
- cmac: 基于分组密码的消息认证码
- cmp: 证书管理协议
- cms: 加密消息语法
- comp: 压缩算法
- conf: 配置文件管理
- crmf: 暂时不确定这个是做什么的
- ct: 证书透明化（Certificate Transparency）
- des: des 对称算法
- dh: 密钥交换协议
- dsa: DSA签名算法
- dso: 动态库管理
- ec: 椭圆曲线算法
- encode_decode: 编码和解码
- engine: 引擎框架
- err: 错误处理
- ess: 增强安全服务（Enhanced Security Services）
- evp: 高层算法接口
- ffc: 有限域加密
- hmac: 基于 hash 的消息鉴别码
- http: http 协议实现
- idea: 国际数据加密算法
- kdf: 密钥派生函数
- lhash: 哈希链表实现
- md2: md2 摘要算法
- md4: md4 摘要算法
- md5: md5 摘要算法
- mdc2: mdc2 摘要算法
- modes: 对称算法的模式
- objects: 对象管理
- ocsp: 在线证书状态协议
- pem: PEM 编解码实现
- pkcs7: 加密签名消息语法标准 PKCS7 实现
- pkcs12: 个人信息交换语法标准 PKCS12 实现
- poly1305: Poly1305 消息认证码
- property: 暂不清楚这个目录下的文件是做什么的
- providers: 安全服务提供者，这个是 openssl3.0+ 最具特色的设计
- rand: 随机数
- rc2: RC2 对称算法
- rc4: RC4 对称算法
- rc5: RC5 对称算法
- ripemd: RACE原始完整性校验消息摘要
- rsa: RSA 非对称算法
- seed: 基于随机种子的对称加密算法
- sha: sha1、sha256、sha512等摘要算法实现
- siphash: SipHash 摘要算法
- sm2: 国密 sm2 椭圆曲线算法
- sm3: 国密 sm3 摘要算法
- sm4: 国密 sm4 分组加密算法
- srp: 暂时不清楚是做什么的
- stack: 栈的实现
- store: 临时存储通道
- ts: 可信时间？
- txt_db: 基于文本的数据库
- ui: 用户接口
- whirlpool: Whirlpool 散列算法
- x509: x509 证书系列标准实现
