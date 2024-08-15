---
title: Openssl 加密算法库编程精要 02 加密一段数据吧！
date: 2022-02-14 11:33:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

调用 Openssl EVP API 来加密数据

<!-- more -->
## 2.1 让我们从 Provider 谈起

相对于老版本的 Openssl，新的 Openssl3.0+ 的一个关键改变是提出了一个 Provider 的概念，我将它命名为安全服务提供者。安全服务提供者集成了相应的算法实现，用户可以采用编程的方式或者在配置文件中指定的方式来使用这些安全服务。Openssl 提供了 5 中不同的标准安全服务，而且我们也可以按照一定的规范自己实现安全服务相关的算法，然后通过 Openssl 提供的顶层的API 来向用户提供安全服务，用户无需再像以前那样调用具体的底层的 API。

Openssl 内置的 5 种安全服务分别为：

1. Default Provider -- 默认安全服务，包含了大多数通用的、主流的算法实现
2. Base Provider -- 基本安全服务，包含了一小部分算法的密钥编解码实现
3. FIPS Provider -- FIPS（联邦信息处理标准） 安全服务，包含了根据FIPS 140-2标准验证的算法实现，这个是美国政府的标准
4. Legacy Provider -- 历史遗留算法的安全服务，包含了一些相对过时的、已于当前证明不安全的算法，如 MD2、RC4 等
5. Null Provider -- 空安全服务，如果你不想让 Default Provider 被自动加载，可以手动加载这个空安全服务替代它

## 2.2 library context -- 库上下文的概念

除了安全服务, Openssl 还提供了一个 Library Context （库上下文）的概念。
库上下文可以被看做是一个安全服务执行操作的作用域，每一个安全服务都被限定在自己的库上下文作用域进行操作。对于复杂的应用来说，有可能要同时用到不同的安全服务，库上下文可以保证不同配置的安全服务同时存在且互不干扰。

库上下文在 Openssl 中被指定为 OSSL_LIB_CTX  类型。应用程序在使用 Openssl 的过程中如果没有显式创建 OSSL_LIB_CTX 对象，那么就需要使用 “default” 类型的库上下文。大多数的 Openssl API 会带有一个 OSSL_LIB_CTX 类型的参数，我们可以显式指定，也可以传 NULL 来使用 Openssl 提供的 “default” 库上下文。

新版的 Openssl 默认库上下文会在需要时自动被创建，所以我们不需要再像使用旧版 Openssl 那样手动初始化所有的算法，而且在应用退出的时候，默认的库上下文也会自动销毁，无需我们手动操作。

## 2.3 获取算法

那么到底如何才能使用 Openssl 提供的算法呢？我们首先从 Openssl 对外提供的头文件中，找到 provider.h 和 evp.h。provider.h 声明了安全服务相关的一系列操作，evp.h 则包含了算法相关的高层 API。然后我们创建一个 main.c：

``` C
#include <stdlib.h>

#include <openssl/provider.h>
#include <openssl/evp.h>

#include "trace/trace.h"

#define SUCCESS 0 /* 成功 */
#define FAILED -1 /* 失败 */

#define ALGO_NAME "SM4-ECB"

/* 定义缓冲区长度 */
#define DATA_BUF_LEN 128

/* 全局缓冲区 */
static unsigned char DATA_BUF[DATA_BUF_LEN] = { 0 };

/* 定义二进制数据块 */
struct BIN_DATA {
    unsigned char *data; /* 数据首地址 */
    int len;             /* 数据长度（字节） */
};

/* 加密数据 */
int encrypt_data(EVP_CIPHER_CTX *ctx, const struct BIN_DATA *in, struct BIN_DATA *enc)
{
    EVP_CIPHER *cipher = NULL;
    int ret = FAILED;
    unsigned char key[16] = "123456788765432";

    int padlen = 0;

    /* 这里仅做简单的判断，实际的生产环境要对各种边界条件做细致的检查，确保应用程序的可靠性 */
    if (!ctx || !in || !enc) {
        return FAILED;
    }

    /* 获取算法 */
    cipher = EVP_CIPHER_fetch(NULL, ALGO_NAME, NULL);
    if (!cipher) {
        return FAILED;
    }

    /* 加密初始化 */
    if (EVP_EncryptInit_ex(ctx, cipher, NULL, key, NULL) != 1) {
        goto end;
    }

    /* 加密 */
    if (EVP_EncryptUpdate(ctx, enc->data, &enc->len, in->data, in->len) != 1) {
        goto end;
    }

    /* 加密结束，处理一些需要填充的数据 */
    if (EVP_EncryptFinal_ex(ctx, enc->data + enc->len, &padlen) != 1) {
        goto end;
    }

    /* 计算最终长度 */
    enc->len += padlen;
    ret = SUCCESS;
end:
    EVP_CIPHER_free(cipher);
    return ret;
}

/* 解密数据 */
int decrypt_data(EVP_CIPHER_CTX *ctx, const struct BIN_DATA *enc, struct BIN_DATA *dec)
{
    EVP_CIPHER *cipher= NULL;
    int ret = FAILED;
    unsigned char key[16] = "123456788765432";

    int padlen = 0;

    if (!ctx || !enc || !dec) {
        return FAILED;
    }

    cipher = EVP_CIPHER_fetch(NULL, ALGO_NAME, NULL);
    if (!cipher) {
        return FAILED;
    }

    if (EVP_DecryptInit_ex(ctx, cipher, NULL, key, NULL) != 1) {
        goto end;
    }

    if (EVP_DecryptUpdate(ctx, dec->data, &dec->len, enc->data, enc->len) != 1) {
        goto end;
    }

    if (EVP_DecryptFinal_ex(ctx, dec->data + dec->len, &padlen) != 1) {
        goto end;
    }

    dec->len += padlen;
    ret = SUCCESS;
end:
    EVP_CIPHER_free(cipher);
    return ret;
}

/**
 * 采用 sm4 算法加密一段字符串
 */
int main(int argc, char *argv[])
{
    int ret = FAILED;
    char data[] = "1234567890";
    int len = sizeof(data);

    /* 原文 */
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

    /* 加载 default provider */
    OSSL_PROVIDER *prov = OSSL_PROVIDER_load(NULL, "default");

    /* 创建上下文 */
    EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();

    /* 打印原文数据 */
    TRACE_BIN("原文数据", in.data, in.len);

    /* 加密数据 */
    ret = encrypt_data(ctx, &in, &enc);
    if (ret != SUCCESS) {
        TRACE("加密失败！\n");
        goto end;
    }

    /* 打印加密数据 */
    TRACE_BIN("加密数据", enc.data, enc.len);

    /* 解密数据 */
    ret = decrypt_data(ctx, &enc, &dec);
    if (ret != SUCCESS) {
        TRACE("解密失败！\n");
        goto end;
    }

    /* 打印解密数据 */
    TRACE_BIN("解密数据", dec.data, dec.len);

    ret = SUCCESS;
end:
    if (ctx) {
        EVP_CIPHER_CTX_free(ctx);
    }

    /* 卸载默认 default provider */
    OSSL_PROVIDER_unload(prov);
    return ret;
}
```

为了测试加解密结果，我们再创建一些列专用用于打印数据的函数，它们在 trace.h 头文件中声明，代码如下所示：

``` C
#ifndef __TRACE_H__
#define __TRACE_H__

#include <stdio.h>

/* 打印日志 */
#define TRACE(log, ...) printf((log), ##__VA_ARGS__)

/* 打印二进制数据 */
#define TRACE_BIN(desc, pt, len) trace_bin((desc), (pt), (len))

/* 打印二进制数据 */
void trace_bin(const char *desc, const void *data, int len);

#endif /* __TRACE_H__ */
```

注：trace_bin 函数用于打印二进制数据块，读者感兴趣可以自己实现，原理非常简单，这里不再展示。

先从 main 函数读起，调用 encrypt_data 函数则将原文数据 “123456789” 加密。输出密文后，利用 decrypt_data 函数将密文解密，再比对解密后的结果是否和原文相同。输出结果如下所示：

``` BASH
原文数据 size:11
------------------------+------------------------
31 32 33 34 35 36 37 38 | 39 30 00 
------------------------+------------------------
加密数据 size:16
------------------------+------------------------
34 b2 cf 92 94 08 c4 10 | 7b ea 27 61 cb 77 8f 64
------------------------+------------------------
解密数据 size:11
------------------------+------------------------
31 32 33 34 35 36 37 38 | 39 30 00 
------------------------+------------------------
```

## 2.4 总结

综上，我们会发现 openssl 的算法基本都是一样的形式：

1. 创建上下文
2. 获取算法
3. 调用 EVP_***Init
4. 调用 EVP_***Update
5. 调用 EVP_***Final
6. 销毁算法和上下文

所以，我们如果想要使用诸如 AES、或者 3DES 等加密算法的话，只需要将算法名称和相应的密钥长度略作调整即可。如果用户想要获取相应的算法信息，如密钥长度或者加密的分组长度，在获取算法实现后，调用 EVP_CIPHER_get_key_length 来获取密钥长度，调用 EVP_CIPHER_get_block_size 来获取分组长度，Openssl 专门为对称算法提供了很多功能性的函数，这里我准备放在下一章节阐述。以上便是本章所有内容。
