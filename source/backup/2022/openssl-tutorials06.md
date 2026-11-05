---
title: Openssl 加密算法库编程精要 06 详解 EVP API 公开密钥密码算法 -- 非对称加密、签名
date: 2022-02-14 15:29:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

简单的加密签名示例

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)

## 6.1 公开密钥算法加解密

公开密钥算法的特点是可以使用公钥加密，用私钥解密。加密主要涉及以下两个接口：

``` C
#include <openssl/evp.h>

/**
 * 加密初始化
 *
 * ctx[in] -- 公钥算法上下文
 * params[in] -- 扩展参数列表
 *
 * 成功，返回 1，否则返回 0
 */
int EVP_PKEY_encrypt_init_ex(EVP_PKEY_CTX *ctx, const OSSL_PARAM params[]);

/**
 * 非对称加密
 *
 * ctx[in] -- 公钥算法上下文
 * out[out] -- 加密后的结果
 * outlen[in/out] -- 加密后的长度或缓冲区的长度
 * in[in] -- 原文数据
 * inlen -- 原文长度
 *
 * 成功，返回密钥对象，否则返回 NULL
 */
int EVP_PKEY_encrypt(EVP_PKEY_CTX *ctx, unsigned char *out, size_t *outlen, const unsigned char *in, size_t inlen);
```

EVP_PKEY_encrypt 这个接口的 out 参数如果传 NULL，那么参数 outlen 就会返回推荐缓冲区的长度；否则如果 out 不为 NULL，那么，outlen 就必须表示为 out 的缓冲区的长度。
和加密一样，解密的接口主要涉及 EVP_PKEY_decrypt_init_ex 和 EVP_PKEY_decrypt，参数完全一致，那么我们简单使用 RSA 算法演示一下非对称加解密，见示例1：

``` C
#include <openssl/evp.h>
#include <trace/trace.h>

/* 定义算法名称 */
#define ALGO_NAME "RSA"

/* 定义缓冲区长度 */
#define DATA_BUF_LEN 1024

/* 全局缓冲区 */
static unsigned char DATA_BUF[DATA_BUF_LEN] = { 0 };

/* 定义二进制数据块 */
struct BIN_DATA {
    unsigned char *data; /* 数据首地址 */
    size_t len;          /* 数据长度（字节） */
};

/* 非对称加密 */
int asym_encrypt(EVP_PKEY_CTX *ctx, struct BIN_DATA *in, struct BIN_DATA *enc)
{
    size_t blen = 0;

    if (!ctx || !in || !enc) {
        return 0;
    }

    /* 先获取加密缓冲区大小 */
    if (EVP_PKEY_encrypt(ctx, NULL, &blen, in->data, in->len) != 1) {
        return 0;
    }

    if (blen > enc->len) {
        TRACE("缓冲区不足！\n");
        return 0;
    }

    /* 非对称加密 */
    if (EVP_PKEY_encrypt(ctx, enc->data, &enc->len, in->data, in->len) != 1) {
        return 0;
    }
    return 1;
}

/* 非对称解密 */
int asym_decrypt(EVP_PKEY_CTX *ctx, struct BIN_DATA *enc, struct BIN_DATA *dec)
{
    size_t blen = 0;

    if (!ctx || !enc || !dec) {
        return 0;
    }

    /* 先获取加密缓冲区大小 */
    if (EVP_PKEY_decrypt(ctx, NULL, &blen, enc->data, enc->len) != 1) {
        return 0;
    }

    if (blen > dec->len) {
        TRACE("缓冲区不足！\n");
        return 0;
    }

    /* 非对称解密 */
    if (EVP_PKEY_decrypt(ctx, dec->data, &dec->len, enc->data, enc->len) != 1) {
        return 0;
    }
    return 1;
}

/* Openssl 这个版本暂时没有实现高层次 api 对 SM2_3 的支持，所以这里用 RSA 算法做演示 */
int main(int argc, char *argv[]) 
{
    EVP_PKEY_CTX *ectx = NULL;
    EVP_PKEY_CTX *dctx = NULL;

    /* 定义原文 */
    char data[] = "12345678901234567890abcdefgABCDEFGIOPBNM12356789";
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

    /* 首先生成非对称密钥对 */
    EVP_PKEY *pair = EVP_PKEY_Q_keygen(NULL, NULL, ALGO_NAME, 2048);

    if (!pair) {
        TRACE("生成密钥失败\n");
        return 0;
    }

    /* 获取非对称算法上下文 */
    ectx = EVP_PKEY_CTX_new_from_pkey(NULL, pair, NULL);
    if (!ectx) {
        TRACE("生成非对称算法上下文失败\n");
        goto end;
    }

    /* 加密前初始化上下文 */
    if (EVP_PKEY_encrypt_init_ex(ectx, NULL) != 1) {
        TRACE("初始化非对称算法上下文失败\n");
        goto end;
    }

    /* 加密 */
    if (asym_encrypt(ectx, &in, &enc) != 1) {
        TRACE("加密失败\n");
        goto end;
    }

    dctx = EVP_PKEY_CTX_new_from_pkey(NULL, pair, NULL);
    if (!dctx) {
        TRACE("生成非对称算法上下文失败\n");
        goto end;
    }

    /* 解密前初始化上下文 */
    if (EVP_PKEY_decrypt_init_ex(dctx, NULL) != 1) {
        TRACE("初始化非对称算法上下文失败\n");
        goto end;
    }

    /* 解密 */
    if (asym_decrypt(dctx, &enc, &dec) != 1) {
        TRACE("解密失败\n");
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
    if (dctx) {
        EVP_PKEY_CTX_free(dctx);
    }

    if (ectx) {
        EVP_PKEY_CTX_free(ectx);
    }

    EVP_PKEY_free(pair);
    return 0;
}
```

结果如下所示：

``` BASH
原文数据 size:49
------------------------+------------------------
31 32 33 34 35 36 37 38 | 39 30 31 32 33 34 35 36
37 38 39 30 61 62 63 64 | 65 66 67 41 42 43 44 45
46 47 49 4f 50 42 4e 4d | 31 32 33 35 36 37 38 39
00 
------------------------+------------------------
加密数据 size:256
------------------------+------------------------
ae c3 d5 4f 2f e7 ce 13 | 39 c9 0f f1 45 34 e4 e3
45 91 5a ac 7b 16 d3 f2 | 5f c8 9e 8a 90 5d ef 6c
85 4e 55 5d 23 62 f4 a9 | 5b 66 ff b7 fb f4 e2 d6
29 1b 63 59 0d b8 38 fb | 59 c9 18 12 fc 13 1e 68
0a dd 4e bb 0b e9 b7 b9 | aa 22 48 cc fb a6 21 5b
d2 53 72 b1 b7 fe 77 c5 | bd 20 4f a8 a8 28 71 82
94 5f 39 32 7c f1 94 9c | e3 be 35 0f 6b f5 dc e5
0a f3 19 94 85 03 d9 bb | 63 76 2d 07 ee fa b6 e7
------------------------+------------------------
49 49 db e9 06 25 39 6c | c2 d8 4d c4 a0 6d 2f a5
43 0f e0 48 4c 2c 5a bd | 4b 7d 4d fd 0c 4d 0f 89
f9 78 39 83 1c c9 34 7d | fd 98 c6 03 c7 10 62 f3
36 27 39 5b 5f e5 6d 1c | 54 39 60 2c 3c 04 a5 a1
21 01 e0 4c 24 85 36 44 | 70 d1 b2 24 01 ed d9 dc
8e 2b 00 10 4d 24 0f e8 | 24 d4 22 62 93 0d be e5
3b a0 6e fd 11 27 64 26 | 6f 0c 2f 1a 71 e4 77 36
02 15 37 79 48 1b 34 c7 | 82 9e 44 89 60 98 65 51
------------------------+------------------------
解密数据 size:49
------------------------+------------------------
31 32 33 34 35 36 37 38 | 39 30 31 32 33 34 35 36
37 38 39 30 61 62 63 64 | 65 66 67 41 42 43 44 45
46 47 49 4f 50 42 4e 4d | 31 32 33 35 36 37 38 39
00 
------------------------+------------------------
```

对于示例 1，我们并没有设置对 RSA 的填充方式，不过 Openssl 提供了设置 RSA 填充方式的接口，接口定义如下：

``` C
#include <openssl/rsa.h>

/**
 * 设置 RSA 填充方式
 *
 * ctx[in] -- 公钥密码算法上下文
 * pad[in] --  填充方式
 *
 * 成功，返回 1，否则返回 0
 */
int EVP_PKEY_CTX_set_rsa_padding(EVP_PKEY_CTX *ctx, int pad);
```

RSA 常用的填充方式有：

- RSA_PKCS1_PADDING --  PKCS#1 v1.5 填充，原文的最大长度为 RSA 模长（字节） - 11
- RSA_PKCS1_OAEP_PADDING -- PKCS#1 v2.0 填充，官方文档表明该填充方式仅被用于加密和解密，原文的最大长度为 RSA 模长（字节）- 42
- RSA_NO_PADDING -- 不填充

除此之外，还有 RSA_X931_PADDING、RSA_PKCS1_PSS_PADDING 以及 RSA_PKCS1_WITH_TLS_PADDING 等填充方式，这里不再讨论。

## 6.2 公开密钥算法签名验签

签名验签的 API 接口和加解密大同小异，接口定义如下：

``` C
#include <openssl/evp.h>
 
/**
 * 签名初始化
 *
 * ctx[in] -- 公钥密码算法上下文
 * params[in] -- 扩展参数
 *
 * 成功，返回 1，否则返回 0
 */
int EVP_PKEY_sign_init_ex(EVP_PKEY_CTX *ctx, const OSSL_PARAM params[]);

/**
 * 签名
 *
 * ctx[in] -- 公钥密码算法上下文
 * sig[out] -- 签名数据
 * siglen[in/out] -- 签名长度或者缓冲区长度
 * tbs[in] -- 原文数据
 * tbslen[in] -- 原文长度
 *
 * 成功，返回 1，否则返回 0
 */
int EVP_PKEY_sign(
    EVP_PKEY_CTX *ctx,
    unsigned char *sig,
    size_t *siglen,
    const unsigned char *tbs,
    size_t tbslen
);

/**
 * 验签初始化
 *
 * ctx[in] -- 公钥密码算法上下文
 * params[in] -- 扩展参数
 *
 * 成功，返回 1，否则返回 0
 */
int EVP_PKEY_verify_init_ex(EVP_PKEY_CTX *ctx, const OSSL_PARAM params[]);

/**
 * 验签
 *
 * ctx[in] -- 公钥密码算法上下文
 * sig[in] -- 签名数据
 * siglen[in] -- 签名长度
 * tbs[in] -- 用于验证的原文数据
 * tbslen[in] -- 用于验证的原文长度
 *
 * 验签成功，返回 1，否则返回 0
 */
int EVP_PKEY_verify(
    EVP_PKEY_CTX *ctx,
    const unsigned char *sig,
    size_t siglen,
    const unsigned char *tbs,
    size_t tbslen
);
```

签名验签的接口和加解密的接口几乎一致，这里不再提供示例程序，读者可以自己测试。以上是本章节的全部内容。
