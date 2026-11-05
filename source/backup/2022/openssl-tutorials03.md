---
title: Openssl 加密算法库编程精要 03 详解 EVP API 对称加密
date: 2022-02-14 11:47:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

Openssl EVP 对称加密指南

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. 《密码学原理与实践（第三版）》，Douglas R.Stinson 冯登国等译
4. 《GB/T 32907-2016 信息安全技术 SM4 分组密码算法》

## 3.1 简介

Openssl 提供的 EVP API 提供了高层级、相对抽象的密码加密接口，是我们最常用的 Openssl 模块之一。它提供了丰富的功能和特性：

- 一系列抽象的密码接口，用户无需考虑底层复杂的实现，使用简单
- 支持对称加密和解密操作，并且支持使用对称算法的各种常用的模式
- 支持消息摘要算法
- 支持非对称加解密和签名验签
- 支持密钥派生
- 支持生成消息认证码

## 3.2 EVP API 的调用规律

我们从上一章节的内容可以知道，调用高层次的 EVP API 基本都遵循固定的规律：    

1. 获取算法实现
2. 创建上下文 CTX
3. 调用 EVP_***Init
4. 调用 EVP_***Update
5. 调用 EVP_***Final
6. 销毁上下文，清理资源

虽然表面看起来确实如此，但是对于很多大部分公开密钥算法而言，由于这些算法的数学原理大相径庭，所以将它们抽象成类似对称算法和摘要算法那样形式的接口非常的困难，所以以上规律在我们使用非对称公开密钥算法时并不适用，这里各位读者需要注意，以后有机会我会对公开密钥算法的调用做专门的说明。

## 3.3 获取算法

在 EVP API 中，和对称算法相关的接口的前缀都是 EVP_CIPHER，我从这里面挑出我认为比较常用的几个接口来说明一下。

首先是获取算法接口：

``` C
#include <openssl/evp.h>

/**
 * 获取算法
 * 
 * ctx[in] -- Openssl 库上下文
 * algorithm[in] -- 算法名称
 * properties[in] -- 属性查询字符串
 *
 * 成功找到算法，返回算法实现，否则返回 NULL
 */
EVP_CIPHER *EVP_CIPHER_fetch(OSSL_LIB_CTX *ctx, const char *algorithm, const char *properties);

/**
 * 销毁算法
 * 
 * cipher[in] -- 算法实现
 */
void EVP_CIPHER_free(EVP_CIPHER *cipher);
```

调用 EVP_CIPHER_fetch 接口第一个参数可以传 NULL，会默认采用全局库上下文。
使用这个接口获取算法在 Openssl 内部被称为“显式获取（Explicit fetching）”，不同于旧版本的直接指定算法实现的方式，新版本在调用 EVP_CIPHER_fetch 获取算法后，需要调用 EVP_CIPHER_free 接口来释放 EVP_CIPHER 对象。
调用 EVP_CIPHER_fetch 接口获取到算法实现后，我们可以通过该实现获取一系列的对称算法信息，比如密钥长度、初始向量长度、分组长度等，接口说明如下：

``` C
#include <openssl/evp.h>

/**
 * 获取分组长度
 * 
 * cipher[in] -- 算法实现
 * 返回分组长度
 */
int EVP_CIPHER_get_block_size(const EVP_CIPHER *cipher);

/**
 * 获取密钥长度
 * 
 * cipher[in] -- 算法实现
 * 返回密钥长度
 */
int EVP_CIPHER_get_key_length(const EVP_CIPHER *cipher);

/**
 * 获取初始化向量长度
 * 
 * cipher[in] -- 算法实现
 * 返回初始化向量长度
 */
int EVP_CIPHER_get_iv_length(const EVP_CIPHER *cipher);
```

具体的调用示例如下 -- 示例1：

``` C
#include <openssl/evp.h>
#include <trace/trace.h>

int main(int argc, char *argv[])
{
    int klen = 0;
    int ilen = 0;
    int blen = 0;

    EVP_CIPHER *sm4 = EVP_CIPHER_fetch(NULL, "SM4-CBC", NULL);

    if (!sm4) {
        return 0;
    }

    klen = EVP_CIPHER_get_key_length(sm4);
    ilen = EVP_CIPHER_get_iv_length(sm4);
    blen = EVP_CIPHER_get_block_size(sm4);

    TRACE("key len = %d, iv len = %d, block len = %d\n", klen, ilen, blen);

    EVP_CIPHER_free(sm4);
    return 0;
}
```

结果如下：

``` BASH
key len = 16, iv len = 16, block len = 16
```

可以看到我们正确获取到了 SM4 算法分组链接模式的算法信息。
在使用获取到的算法实现进行运算之前，我们需要获取到加密上下文 -- cipher context，对应 Openssl 中则是 EVP_CIPHER_CTX 对象，创建和销毁上下文的接口说明如下：

``` C
#include <openssl/evp.h>

/**
 * 创建加密上下文
 * 
 * 返回加密上下文对象
 */
EVP_CIPHER_CTX *EVP_CIPHER_CTX_new(void);

/**
 * 销毁加密上下文
 * 
 * ctx[in] -- 加密上下文对象
 */
void EVP_CIPHER_CTX_free(EVP_CIPHER_CTX *ctx);
```

加密上下文相当于执行加密运算时的环境，里面维护了加解密运算时需要的临时数据，在执行加密运算初始化前，必须首先创建加密上下文，我们还是以 SM4-CBC 算法为例，先看一下完整的示例程序示例 2：

``` C
#include <openssl/evp.h>
#include <trace/trace.h>
 
/* 加密和解密标记 */
#define ENCRYPT 1
#define DECRYPT 0
 
/* 定义缓冲区长度 */
#define DATA_BUF_LEN 256
 
/* 全局缓冲区 */
static unsigned char DATA_BUF[DATA_BUF_LEN] = { 0 };
 
/* 定义二进制数据块 */
struct BIN_DATA {
    unsigned char *data; /* 数据首地址 */
    int len;             /* 数据长度（字节） */
};
 
/* 加密或解密 */
int cipher(
    EVP_CIPHER_CTX *ctx, EVP_CIPHER *cipher, const struct BIN_DATA *in, struct BIN_DATA *out, int enc)
{
    unsigned char key[] = {
        0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
        0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x00
    };
 
    unsigned char iv[] = {
        0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
        0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0xFF
    };
 
    int padlen = 0;
 
    if (!ctx || !cipher || !in || !out) {
        return 0;
    }
 
    if (EVP_CipherInit_ex2(ctx, cipher, key, iv, enc, NULL) != 1) {
        return 0;
    }
 
    if (EVP_CipherUpdate(ctx, out->data, &out->len, in->data, in->len) != 1) {
        return 0;
    }
 
    if (EVP_CipherFinal_ex(ctx, out->data + out->len, &padlen) != 1) {
        return 0;
    }
 
    /* 计算最终长度 */
    out->len += padlen;
    return 1;
}
 
int main(int argc, char *argv[])
{
    char data[] = "12345678901234567890abcdefgABCDEFGIOPBNM1235678";
    int len = sizeof(data);
 
    /* 原文数据 */
    struct BIN_DATA in = {
        (unsigned char *)data,
        len
    };
 
    /* 加密缓冲区 */
    struct BIN_DATA enc = {
        DATA_BUF,
        DATA_BUF_LEN / 2
    };
 
    /* 解密缓冲区 */
    struct BIN_DATA dec = {
        DATA_BUF + DATA_BUF_LEN / 2,
        DATA_BUF_LEN / 2
    };
 
    /* 加密上下文 */
    EVP_CIPHER_CTX *ctx = NULL;
 
    /* 获取算法 */
    EVP_CIPHER *sm4 = EVP_CIPHER_fetch(NULL, "SM4-CBC", NULL);
 
    if (!sm4) {
        return 0;
    }
 
    /* 创建加密上下文 */
    ctx = EVP_CIPHER_CTX_new();
    if (!ctx) {
        goto end;
    }
 
    /* 加密 */
    if (cipher(ctx, sm4, &in, &enc, ENCRYPT) != 1) {
        TRACE("加密失败！\n");
        goto end;
    }
 
    /* 重置上下文 */
    if (EVP_CIPHER_CTX_reset(ctx) != 1) {
        TRACE("上下文重置失败！\n");
        goto end;
    }
 
    /* 解密 */
    if (cipher(ctx, sm4, &enc, &dec, DECRYPT) != 1) {
        TRACE("解密失败！\n");
        goto end;
    }
 
    /* 打印信息 */
 
    /* 打印原文数据 */
    TRACE_BIN("原文数据", in.data, in.len);
     
    /* 打印加密数据 */
    TRACE_BIN("加密数据", enc.data, enc.len);
 
    /* 打印解密数据 */
    TRACE_BIN("解密数据", dec.data, dec.len);

end:
    if (ctx) {
        EVP_CIPHER_CTX_free(ctx);
    }

    if (sm4) {
        EVP_CIPHER_free(sm4);
    }

    return 0;
}
```

这里我们单独定义了一个名为 cipher 的函数，cipher 函数中首先定义了密钥 key 和初始化向量 iv，这里我提供几个生成 key 和 iv 的方法：

1. 拍脑门随机想两个对应长度的值
2. 直接调用随机数生成器 RAND_bytes 生成两个对应的长度的值
3. 创建 EVP_CIPHER_CTX 后，调用 EVP_CIPHER_CTX_rand_key 接口生成随机密钥， iv 可以采用 1 或者 2 的方式生成

我们采用第一种方法… 然后分别调用了 EVP_CipherInit_ex2、EVP_CipherUpdate 和 EVP_CipherFinal_ex 三个接口来完成对称加密解密操作，这三个接口定义如下：

``` C
#include <openssl/evp.h>

/**
 * 分组加解密初始化
 * 
 * ctx[in] -- 加密上下文
 * type[in] -- 算法实现
 * key[in] -- 对称密钥
 * iv[in] -- 初始化向量
 * enc[in] -- 加密或者解密标志，传 1 为加密，传 0 为解密
 * params[in] -- 扩展参数
 *
 * 调用成功，返回 1，否则返回 0
 */
int EVP_CipherInit_ex2(
    EVP_CIPHER_CTX *ctx,
    const EVP_CIPHER *type,
    const unsigned char *key,
    const unsigned char *iv,
    int enc,
    const OSSL_PARAM params[]
);

/**
 * 分组加解密
 * 
 * ctx[in] -- 加密上下文
 * out[out] -- 存放加密或者解密结果的内存首地址
 * outl[out] -- 结果长度
 * in[in] -- 存放原文数据的内存首地址
 * inl[in] -- 原文数据长度
 *
 * 调用成功，返回 1，否则返回 0
 */
int EVP_CipherUpdate(
    EVP_CIPHER_CTX *ctx,
    unsigned char *out,
    int *outl,
    const unsigned char *in,
    int inl
);

/**
 * 加解密收尾
 * 
 * ctx[in] -- 加密上下文对象
 * outm[out] -- 存放剩余加密或者解密结果的内存首地址
 * outl[out] -- 收尾时处理的数据长度
 *
 * 调用成功，返回 1，否则返回 0
 */
int EVP_CipherFinal_ex(EVP_CIPHER_CTX *ctx, unsigned char *outm, int *outl);
```

可以参考之前的示例 2，在 cipher 函数中，我们通过对 EVP_CipherInit_ex2 的 enc 参数传入不同的值来控制程序的逻辑是加密还是解密，而且对于加解密所需要的密钥和初始化向量（如果算法模式是 ECB 模式则为 NULL）也是在调用该接口传入。

默认情况下，Openssl 会对原文的数据进行 PKCS7 填充以满足加密算法的应用需求。大多数的分组加密算法的块长度都是 8 字节或者是 16 字节，在加密数据之前，如果原文长度不是分组长度的整数倍，那么就需要用当前缺失的字节数来作为填充内容，填充至原文末尾，直至原文的长度为加密块长度的整数倍；如果当前原文长度恰好为加密块的整数倍，那么仍然需要在原文后添加一个块长度的填充数据，以该块长度作为填充内容。为了演示填充操作，我们将示例 2 的程序稍加修改，如下示例 3 所示：

``` C
……（同示例2）

int main(int argc, char *argv[])
{
    char data[] = "12345678901234567890abcdefgABCDEFGIOPBNM1235678";
    int len = sizeof(data);

    /* 原文数据 */
    struct BIN_DATA in = {
        (unsigned char *)data,
        len
    };

    /* 加密缓冲区 */
    struct BIN_DATA enc = {
        DATA_BUF,
        DATA_BUF_LEN / 2
    };

    /* 解密缓冲区 */
    struct BIN_DATA dec = {
        DATA_BUF + DATA_BUF_LEN / 2,
        DATA_BUF_LEN / 2
    };

    /* 加密上下文 */
    EVP_CIPHER_CTX *ctx = NULL;

    /* 获取算法 */
    EVP_CIPHER *sm4 = EVP_CIPHER_fetch(NULL, "SM4-CBC", NULL);

    if (!sm4) {
        return 0;
    }

    /* 创建加密上下文 */
    ctx = EVP_CIPHER_CTX_new();
    if (!ctx) {
        goto end;
    }

    /* 加密 */
    if (cipher(ctx, sm4, &in, &enc, ENCRYPT) != 1) {
        TRACE("加密失败！\n");
        goto end;
    }

    /* 重置上下文 */
    if (EVP_CIPHER_CTX_reset(ctx) != 1) {
        TRACE("上下文重置失败！\n");
        goto end;
    }

    /* 取消填充 */
    EVP_CIPHER_CTX_set_padding(ctx, 0);

    /* 解密 */
    if (cipher(ctx, sm4, &enc, &dec, DECRYPT) != 1) {
        TRACE("解密失败！\n");
        goto end;
    }

    ……（以下内容同示例 2）
}
```

运算结果为：

``` BASH
原文数据 size:48
------------------------+------------------------
31 32 33 34 35 36 37 38 | 39 30 31 32 33 34 35 36
37 38 39 30 61 62 63 64 | 65 66 67 41 42 43 44 45
46 47 49 4f 50 42 4e 4d | 31 32 33 35 36 37 38 00
------------------------+------------------------
加密数据 size:64
------------------------+------------------------
7a 7b 69 88 23 12 70 e7 | f0 d9 48 d9 dc 94 68 cf
01 19 d2 d4 92 3f f6 2f | dc 30 a6 b5 d7 b7 f2 47
87 eb 7b 08 a1 4d c2 f1 | d2 f5 28 c3 d2 2a e1 44
77 16 70 d5 3c ce d6 7f | 20 fb 78 45 b4 ea f8 78
------------------------+------------------------
解密数据 size:64
------------------------+------------------------
31 32 33 34 35 36 37 38 | 39 30 31 32 33 34 35 36
37 38 39 30 61 62 63 64 | 65 66 67 41 42 43 44 45
46 47 49 4f 50 42 4e 4d | 31 32 33 35 36 37 38 00
10 10 10 10 10 10 10 10 | 10 10 10 10 10 10 10 10
------------------------+------------------------
```

我们在这里使用了名为 EVP_CIPHER_CTX_set_padding 的函数，这个函数的作用是启用和禁用填充。在调用加密原文数据之后，我们禁用填充，那么解密的时候就会将填充数据打印出来（默认情况下解密时会反填充恢复数据，但是禁用填充功能后则不做任何操作直接输出解密数据），我们可以看到填充长度等于 SM4加密块长度，均为 16 字节，填充内容也是 0x10。

以上是加密原文恰好是 48 字节的情况，然后我们将 data 的内容改为 "12345678901234567890abcdefgABCDEFGIOPBNM12356789"，包含‘\0’ 的情况下恰好 49 个字节，然后运行程序，结果如下：

``` BASH
原文数据 size:49
------------------------+------------------------
31 32 33 34 35 36 37 38 | 39 30 31 32 33 34 35 36
37 38 39 30 61 62 63 64 | 65 66 67 41 42 43 44 45
46 47 49 4f 50 42 4e 4d | 31 32 33 35 36 37 38 39
00 
------------------------+------------------------
加密数据 size:64
------------------------+------------------------
7a 7b 69 88 23 12 70 e7 | f0 d9 48 d9 dc 94 68 cf
01 19 d2 d4 92 3f f6 2f | dc 30 a6 b5 d7 b7 f2 47
1e eb b0 4c 16 6a 1f 7b | ca fc dd e8 ed f5 8d 07
52 a9 42 5b de 37 72 70 | d9 57 64 5d 26 15 0e 95
------------------------+------------------------
解密数据 size:64
------------------------+------------------------
31 32 33 34 35 36 37 38 | 39 30 31 32 33 34 35 36
37 38 39 30 61 62 63 64 | 65 66 67 41 42 43 44 45
46 47 49 4f 50 42 4e 4d | 31 32 33 35 36 37 38 39
00 0f 0f 0f 0f 0f 0f 0f | 0f 0f 0f 0f 0f 0f 0f 0f
------------------------+------------------------
```

我们可以看到 Openssl 自动在原文数据的末尾填充了 15 个 0x0F，恰好是原文长度和分组长度整数倍（4 倍 64 字节）的差值。

以上是我们对常用的分组密码算法 API 的讨论，在实际生产环境中，应该根据自己的项目的需求选择不同的分组密码模式，对我来说一般在加密较小的数据或者密钥时直接采用 ECB 模式，简单快捷，加密大一点的文件或者数据时采用 CBC 模式。CFB、OFB和 CTR 等模式笔者基本没有接触或者使用过，这里不做讨论。
以上就是本章的内容，下一章我们将讨论摘要算法。
