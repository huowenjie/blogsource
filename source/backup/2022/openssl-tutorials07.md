---
title: Openssl 加密算法库编程精要 07 详解 EVP API 公开密钥密码算法 -- 密钥编解码
date: 2022-02-14 15:37:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

介绍几种密钥编解码方法

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. 《ASN.1编码规则详解（最全最经典）》 -- 无名氏译
4. 《ASN.1  Communication between Heterogeneous Systems》-- Olivier Dubuisson
5. 《ASN.1 Complete》 -- J.Larmouth
6. 《GBT 16262.1-2006 信息技术 抽象语法记法一(ASN.1) 第1部分基本记法规范》
7. 《GBT 16262.1-2006 信息技术 抽象语法记法一(ASN.1) 第2部分信息客体规范》
8. 《GBT 16262.1-2006 信息技术 抽象语法记法一(ASN.1) 第3部分约束规范》
9. 《GBT 16262.1-2006 信息技术 抽象语法记法一(ASN.1) 第4部分参数化》
10. 《GBT 16263.1-2006 信息技术 ASN.1 编码规则 第1部分：BER、DER、CER规范》
11. 《GBT 16263.2-2006 信息技术 ASN.1 编码规则 第2部分：紧缩编码规则（PER）规范》

## 7.1 ASN.1 标准简介

ASN.1(Abstract Syntax Notation dot one)，抽象语法记法1，是定义抽象数据类型规格形式的标准，是用于描述数据的表示、编码、传输、解码的灵活的记法。它提供了一套正式的、无歧义和精确的规则，以描述独立于特定计算机硬件的对象结构。ASN.1 特别适合表示现代通信应用中那些复杂的、变化的以及可扩展的数据结构。

ASN.1 本身只定义了表示信息的抽象语法，并没有限定其编码方式。各种 ASN.1 编码规则提供了由 ASN.1 描述其抽象句法数据值的传送方法，常见的 ASN.1 编码方式有如下几种：

- BER 编码，Basic Encoding Rules，基本编码规则；
- CER 编码，Canonical Encoding Rules，规范编码规则；
- DER 编码，Distinguished Encoding Rules，可分辨编码规则（或者叫唯一编码规则）；
- PER 编码，Packed Encoding Rules，压缩编码规则；
- XER 编码，XML Encoding Rules，XML 编码规则。

一个由 ASN.1 编码的源文件可以很方便地解码为 C 或者 C++ 这样语言的数据结构。绝大多数操作系统上的工具都支持 ASN.1，其应用非常广泛，包括并不限于视频、音频、数据流以及各种网络协议等。同时 ASN.1 也是一种用于描述结构化客体的结构和内容的语言，大量密码行业的标准都用 ASN.1 来描述其定义的数据结构。比如在 RFC3447《Public-Key Cryptography Standards (PKCS) #1: RSA Cryptography Specifications Version 2.1》标准中采用 ASN.1 描述 RSA 公钥和私钥的数据格式，如下示例 1 所示：

``` C
RSAPublicKey ::= SEQUENCE {
    modulus INTEGER, -- n
    publicExponent INTEGER -- e
}

RSAPrivateKey ::= SEQUENCE {
    version Version,
    modulus INTEGER, -- n
    publicExponent INTEGER, -- e
    privateExponent INTEGER, -- d
    prime1 INTEGER, -- p
    prime2 INTEGER, -- q
    exponent1 INTEGER, -- d mod (p-1)
    exponent2 INTEGER, -- d mod (q-1)
    coefficient INTEGER, -- (inverse of q) mod p
    otherPrimeInfos OtherPrimeInfos OPTIONAL
}

Version ::= INTEGER { two-prime(0), multi(1) } (CONSTRAINED BY {-- version must be multi if otherPrimeInfos present --})
OtherPrimeInfos ::= SEQUENCE SIZE(1..MAX) OF OtherPrimeInfo
OtherPrimeInfo ::= SEQUENCE {
　　prime INTEGER, -- ri
　　exponent INTEGER, -- di
　　coefficient INTEGER -- ti
}
```

其中，对于公钥结构，SEQENCE 代表顺序结构，相当于 C 语言中的结构体，INTEGER 代表整数；私钥结构的第一个成员是一个名为 Version，实际上它是一个基本类型 INTEGER，并且在 Version 成员上还定义了一个自定义约束，这个约束表明如果当前 RSA 算法仅使用了两个大素数进行运算的话，版本号是 0，否则如果使用了多余两个素数，则版本号是 1。私钥的最后一个成员 OtherPromeInfos 被 OPTIONAL 修饰，表明它是可选的，OtherPrimeInfos 是一个 OtherPrimeInfo 对象的无序列表，每个 OherPrimerInfo 内部分别包含  prime、exponent 和 coefficient 成员。

ASN.1 需要学习的内容远不止如此，我会在下一节专门讲述 Openssl ASN1 模块的功能，这里我整理了一部分关于 ASN.1 的资料，放在本文的开头，感兴趣的读者可以参考。

## 7.2 密钥的 DER 编码和解码

Openssl 提供了密钥的编解码接口，接口定义如下：

``` C
#include <openssl/evp.h>

/**
 * 私钥解码
 *
 * type[in] -- 算法类型
 * a[out] -- 解码后的密钥对象
 * pp[in] -- 待解码的密钥数据首地址
 * length[in] -- 数据长度
 * 
 * 成功，返回密钥对象，否则返回 NULL
 */
EVP_PKEY *d2i_PrivateKey(int type, EVP_PKEY **a, const unsigned char **pp, long length);

/**
 * 公钥解码
 *
 * type[in] -- 算法类型
 * a[out] -- 解码后的密钥对象
 * pp[in] -- 待解码的密钥数据首地址
 * length[in] -- 数据长度
 * 
 * 成功，返回密钥对象，否则返回 NULL
 */
EVP_PKEY *d2i_PublicKey(int type, EVP_PKEY **a, const unsigned char **pp, long length);

/**
 * 私钥编码
 *
 * a[in] -- 密钥对象
 * pp[out] -- 存储编码数据的缓冲区地址
 * 
 * 成功，返回编码数据的长度，否则返回一个负数
 */
int i2d_PrivateKey(const EVP_PKEY *a, unsigned char **pp);

/**
 * 公钥编码
 *
 * a[in] -- 密钥对象
 * pp[out] -- 存储编码数据的缓冲区地址
 * 
 * 成功，返回编码数据的长度，否则返回一个负数
 */
int i2d_PublicKey(const EVP_PKEY *a, unsigned char **pp);
```

可以看到，Openssl 中对于 der 编解码有一个约定俗成的命名规则：如果是 der 编码，函数前缀则是 i2d(internal to der)，如果是 der 解码，则函数前缀则是 d2i(der to internal)，以上四个函数均是这个规律。
对于密钥编码的两个接口，Openssl 允许第二个参数 pp 传 NULL，这样可以预先计算出编码后的数据大小，适合动态分配内存的场景；如果 pp 参数不为 NULL，那么必须保证存放出参的缓冲区足够放得下编码后的数据，如下示例 2 所示：

``` C
#include <stdlib.h>
#include <string.h>
#include <openssl/evp.h>

#include <trace/trace.h>

struct BIN_DATA {
    size_t len;
    unsigned char *data;
};

int main(int argc, char *argv[]) 
{
    EVP_PKEY_CTX *ctx = NULL;
    
    struct BIN_DATA pub = { 0, NULL };
    struct BIN_DATA prv = { 0, NULL };

    unsigned char *p = NULL;

    /* 首先生成非对称密钥 */
    EVP_PKEY *pair = EVP_PKEY_Q_keygen(NULL, NULL, "RSA", 1024);

    if (!pair) {
        TRACE("生成密钥失败\n");
        return 0;
    }

    /* 获取非对称算法上下文 */
    ctx = EVP_PKEY_CTX_new_from_pkey(NULL, pair, NULL);
    if (!ctx) {
        TRACE("生成非对称算法上下文失败\n");
        goto end;
    }

    pub.len = (size_t)i2d_PublicKey(pair, NULL);
    prv.len = (size_t)i2d_PrivateKey(pair, NULL);

    pub.data = malloc(pub.len);
    prv.data = malloc(prv.len);

    memset(pub.data, 0, pub.len);
    memset(prv.data, 0, prv.len);

    p = pub.data;
    if (i2d_PublicKey(pair, &p) != (int)pub.len) {
        TRACE("公钥转码失败\n");
        goto end;
    }

    p = prv.data;
    if (i2d_PrivateKey(pair, &p) != (int)prv.len) {
        TRACE("私钥转码失败\n");
        goto end;
    }

    TRACE_BIN("pubkey der", pub.data, (int)pub.len);
    TRACE_BIN("prvkey der", prv.data, (int)prv.len);

    save_data("pubkey.der", pub.data, (int)pub.len);
    save_data("prvkey.der", prv.data, (int)prv.len);

end:
    if (pub.data) {
        free(pub.data);
    }

    if (prv.data) {
        free(prv.data);
    }

    if (ctx) {
        EVP_PKEY_CTX_free(ctx);
    }

    EVP_PKEY_free(pair);
    return 0;
}
```

测试程序采用了 RSA1024 来作为测试算法，save_data 函数则是一个快速保存文件的函数，生成的密钥在转码之后分别保存在 pubkey.der 和 prvkey.der 两个文件中。保存完毕后，在测试程序的当前目录下我们会找到 pubkey.der 和 prvkey.der 这两个文件，然后我们执行示例2来对密钥读入并解码打印，如下示例 3 所示：

``` C
#include <stdlib.h>
#include <string.h>
#include <openssl/evp.h>

#include <trace/trace.h>

struct BIN_DATA {
    size_t len;
    unsigned char *data;
};

int main(int argc, char *argv[]) 
{
    struct BIN_DATA pub = { 0, NULL };
    struct BIN_DATA prv = { 0, NULL };

    EVP_PKEY *pub_key = NULL;
    EVP_PKEY *prv_key = NULL;

    const unsigned char *p = NULL;

    /* 读取密钥文件 */
    read_data("pubkey.der", NULL, (int *)&pub.len);
    read_data("prvkey.der", NULL, (int *)&prv.len);

    if (pub.len <= 0 || prv.len <= 0) {
        TRACE("读取密钥失败\n");
        return 0;
    }

    pub.data = malloc(pub.len);
    prv.data = malloc(prv.len);

    memset(pub.data, 0, pub.len);
    memset(prv.data, 0, prv.len);

    read_data("pubkey.der", pub.data, (int *)&pub.len);
    read_data("prvkey.der", prv.data, (int *)&prv.len);

    p = pub.data;
    d2i_PublicKey(EVP_PKEY_RSA, &pub_key, &p, (long)pub.len);
    if (!pub_key) {
        TRACE("解码公钥失败\n");
        goto end;
    }

    p = prv.data;
    d2i_PrivateKey(EVP_PKEY_RSA, &prv_key, &p, (long)prv.len);
    if (!prv_key) {
        TRACE("解码私钥失败\n");
        goto end;
    }

    /* 打印解码后的密钥 */
    EVP_PKEY_print_public_fp(stdout, pub_key, 0, NULL);
    EVP_PKEY_print_private_fp(stdout, prv_key, 0, NULL);

end:
    if (pub_key) {
        EVP_PKEY_free(pub_key);
    }

    if (prv_key) {
        EVP_PKEY_free(prv_key);
    }

    if (pub.data) {
        free(pub.data);
    }

    if (prv.data) {
        free(prv.data);
    }

    return 0;
}
```

示例 3 程序中调用了 read_data 函数，它是一个读取文件的函数，我们用它来读取在示例 2 中保存的两个密钥文件，并将密钥数据保存在内存中，利用 d2i_* 的这两个接口将编码数据转换为内部数据，然后再打印出密钥的数据，结果如下所示：

``` BASH
RSA Public-Key: (1024 bit)
Modulus:
    00:b3:61:50:b9:32:3c:14:02:79:ea:84:1f:0d:e8:
    70:e3:ca:2c:7f:a0:e9:69:ac:e3:81:b8:46:16:51:
    64:c5:13:46:1d:64:26:c8:57:51:4b:8b:6a:c6:4f:
    1d:e6:07:33:21:5a:6d:81:78:cb:53:c3:cf:7a:6b:
    b4:d3:b5:5f:c2:7f:59:9d:79:df:40:f7:ed:80:2a:
    e6:b1:5f:31:37:8a:bf:48:ce:08:38:06:09:f0:7b:
    bb:50:4e:69:a0:31:04:96:44:a0:84:38:e9:9e:a5:
    67:6f:92:db:51:52:53:ca:0f:1f:bb:dc:17:fd:ab:
    82:4a:e7:b0:43:c9:0f:0f:89
Exponent: 65537 (0x10001)
Private-Key: (1024 bit, 2 primes)
modulus:
    00:b3:61:50:b9:32:3c:14:02:79:ea:84:1f:0d:e8:
    70:e3:ca:2c:7f:a0:e9:69:ac:e3:81:b8:46:16:51:
    64:c5:13:46:1d:64:26:c8:57:51:4b:8b:6a:c6:4f:
    1d:e6:07:33:21:5a:6d:81:78:cb:53:c3:cf:7a:6b:
    b4:d3:b5:5f:c2:7f:59:9d:79:df:40:f7:ed:80:2a:
    e6:b1:5f:31:37:8a:bf:48:ce:08:38:06:09:f0:7b:
    bb:50:4e:69:a0:31:04:96:44:a0:84:38:e9:9e:a5:
    67:6f:92:db:51:52:53:ca:0f:1f:bb:dc:17:fd:ab:
    82:4a:e7:b0:43:c9:0f:0f:89
publicExponent: 65537 (0x10001)
privateExponent:
    18:c1:c4:97:5a:c4:89:ea:71:93:19:5b:03:db:61:
    c1:3e:84:f7:b4:68:a2:8a:16:f8:2f:4b:95:06:f4:
    c6:72:4b:8a:00:e9:8d:5a:e7:c0:6a:64:79:2c:30:
    2f:30:2d:31:5c:3e:a2:d0:de:17:18:7e:49:22:16:
    59:e5:bd:6a:6a:55:3c:fb:2a:fb:85:e2:f3:56:72:
    a0:64:26:ed:55:3e:8f:94:e9:de:cc:03:ea:64:20:
    53:53:02:f6:25:aa:0d:47:c8:22:26:5e:12:e3:c7:
    19:32:d3:66:da:73:bd:2c:20:42:f6:74:fc:62:6c:
    09:39:ce:a5:79:66:af:11
prime1:
    00:e2:75:bb:50:7f:e8:93:69:0c:d9:9a:1e:c1:84:
    1b:34:d2:7c:c5:e5:f1:41:4d:42:28:a6:c6:e8:47:
    75:de:95:77:95:29:0b:da:58:38:2c:eb:b3:41:68:
    16:4b:f2:39:bb:57:f6:d4:1b:e9:97:0c:5f:b4:09:
    35:5a:31:e6:b5
prime2:
    00:ca:c7:6e:1b:ac:c3:36:2c:0d:d7:1d:ca:e3:6f:
    15:d2:34:c6:b9:c7:51:ab:1a:4b:a8:ca:1d:95:64:
    29:cb:7a:e5:3c:34:97:a1:45:8a:88:63:d2:00:17:
    2b:93:5e:9a:43:f2:5c:f0:e4:15:00:9b:bd:ce:ed:
    11:e2:7a:16:05
exponent1:
    00:da:97:82:03:a6:33:bd:76:bd:6c:9e:13:e9:ff:
    b6:b3:3a:2a:2e:6c:52:80:12:2f:36:46:25:e1:b8:
    78:d2:2d:bc:8c:42:5e:aa:98:55:41:27:12:94:a4:
    00:41:b6:c2:7b:4f:e1:75:c4:ab:a9:9d:cc:13:60:
    80:1b:5b:e7:b1
exponent2:
    14:d9:ea:fd:97:87:3f:43:ca:6c:8b:58:b8:88:4c:
    b3:1f:d0:2b:7c:4e:6e:8c:b6:a8:f5:97:93:2c:08:
    8c:2e:e7:f1:87:ea:eb:9f:6d:fe:56:5d:5a:bb:07:
    35:11:2e:45:bc:5f:48:39:fb:da:e3:28:e2:65:48:
    48:84:8b:4d
coefficient:
    04:49:14:53:cf:c2:54:4b:d8:e6:33:88:36:ea:5c:
    ad:de:10:bd:73:27:d6:b1:6c:8d:7f:7e:88:6b:de:
    ab:25:64:3f:d1:1b:75:5a:86:4e:a8:59:58:4a:57:
    91:03:e1:7d:bd:47:cb:c4:81:02:fb:fa:8a:32:10:
    ff:35:10:b7
```

以上是本章的全部内容。
