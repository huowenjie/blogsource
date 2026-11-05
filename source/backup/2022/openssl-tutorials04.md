---
title: Openssl 加密算法库编程精要 04 详解 EVP API 消息摘要
date: 2022-02-14 11:59:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

Openssl EVP 消息摘要指南

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. 《应用密码学 协议、算法与 C 源程序》，Bruce Schneier 著 吴世忠 祝世雄 张文政等译
4. 《GB/T 32905--2016 信息安全技术 SM3 密码杂凑算法》

## 4.1 消息摘要的概念

消息摘要有好几个名字，比如单项散列函数，Hash 函数，它是一个将可变长度的输入串转换为一个固定长度的输出串的函数。大多数消息摘要算法都是公开的，它的安全性依赖于它的单向性，如果仅获取到消息摘要的结果，想要从结果反推出原文几乎是不可能的事情。并且对于输入串的细微改变，都会引发输出串的雪崩式变化，所以消息摘要一般用于校验数据完整性、是否经过篡改或者实现数字签名。

常用的消息摘要算法有 MD 家族的 MD2、MD4、MD5，SHA 家族的 SHA1、SHA256、SHA384、SHA512 等，还有国密算法 SM3 也是一个设计精巧的摘要函数，我们接下来就以 SM3 算法为例来展示 EVP MD-API 的用法。

## 4.2 消息摘要 API 调用

使用消息摘要的方法比较简单，基本步骤和使用对称加密接口的方法类似。

Openssl 提供了一个快速计算摘要的接口 EVP_Q_digest，可以方便的计算长度较小的原文数据，如下所示：

``` C
#include <openssl/evp.h>

/**
 * 快速一次性摘要
 *
 * libctx[in] -- Openssl 库上下文
 * name[in] -- 算法名称
 * propq[in] -- 扩展属性
 * data[in] -- 原文数据
 * datalen[in] -- 原文长度
 * md[out] -- 摘要数据
 * mdlen[in/out] -- 摘要长度
 *
 * 调用成功，返回 1，否则返回 0
 */
int EVP_Q_digest(
    OSSL_LIB_CTX *libctx,
    const char *name,
    const char *propq,
    const void *data,
    size_t datalen,
    unsigned char *md,
    size_t *mdlen
);
```

这个函数并不需要我们手动获取算法实现，只需要提供算法的名称即可，看下面的示例1：

``` C
#include <stdlib.h>
#include <string.h>
#include <openssl/evp.h>

#include <trace/trace.h>

int main(int argc, char *argv[])
{
    /* 定义原文数据和输出缓冲区 */
    unsigned char data[] = "12345678";
    unsigned char *out = NULL;

    size_t mdlen = EVP_MAX_MD_SIZE;

    /* 为输出缓冲区申请内存 */
    out = malloc(EVP_MAX_MD_SIZE);
    if (!out) {
        return 0;
    }
    memset(out, 0, EVP_MAX_MD_SIZE);

    if (EVP_Q_digest(NULL, "SM3", NULL, data, sizeof(data), out, (unsigned int *)&mdlen) != 1) {
        TRACE("摘要运算失败");
        goto end;
    }

    TRACE_BIN("Digest", out, mdlen);

end:
    free(out);
    return 0;
}
```

结果如下：

``` BASH
Digest size:32
------------------------+------------------------
d3 70 75 73 6f 58 21 70 | 18 a9 ac ba a1 5f 7d 63
58 a9 dd 21 26 cd b5 34 | be 38 7a ef 0c 58 83 40
------------------------+------------------------
```

对于示例1，我们直接动态申请了一个 EVP_MAX_MD_SIZE 长度的缓冲区，然后调用 EVP_Q_digest 函数求摘要值，值得注意的是，EVP_Q_digest md 和 mdlen 两个参数不能为 NULL，否则会引起内存访问错误，导致程序崩溃（至少在这个 3.0.1 版本的 Openssl 会出现这种情况），如果想要精确获取摘要长度，可以先调用接口 EVP_MD_get_size 接口获取摘要长度，然后再申请内存。

对于分组摘要，调用略为复杂，但是基本上和对称加密的形式相似。首先是获取算法实现，接口定义如下所示：

``` C
#include <openssl/evp.h>

/**
 * 获取摘要算法
 *
 * ctx[in] -- Openssl 库上下文
 * algorithm[in] -- 算法名称
 * properties[in] -- 属性查询字符串
 *
 * 成功找到算法，返回算法实现，否则返回 NULL
 */
EVP_MD *EVP_MD_fetch(OSSL_LIB_CTX *ctx, const char *algorithm, const char *properties);

/**
 * 销毁摘要算法
 *
 * md[in] -- 算法实现
 */
void EVP_MD_free(EVP_MD *md);
```

然后是创建消息摘要算法上下文：

``` C
#include <openssl/evp.h>

/**
 * 创建摘要上下文
 *
 * 返回摘要上下文对象
 */
EVP_MD_CTX *EVP_MD_CTX_new(void);
 
/**
 * 销毁摘要上下文
 *
 * ctx[in] -- 摘要上下文对象
 */
void EVP_MD_CTX_free(EVP_MD_CTX *ctx);
```

最后是执行摘要运算，相关的接口如下所示：

``` C
#include <openssl/evp.h>

/**
 * 分组摘要初始化
 *
 * ctx[in] -- 摘要上下文
 * type[in] -- 算法实现
 * params[in] -- 扩展参数
 *
 * 调用成功，返回 1，否则返回 0
 */
int EVP_DigestInit_ex2(EVP_MD_CTX *ctx, const EVP_MD *type, const OSSL_PARAM params[]);

/**
 * 分组摘要运算
 *
 * ctx[in] -- 摘要上下文
 * d[in] -- 原文数据
 * cnt[out] -- 原文长度
 *
 * 调用成功，返回 1，否则返回 0
 */
int EVP_DigestUpdate(EVP_MD_CTX *ctx, const void *d, size_t cnt);

/**
 * 分组摘要收尾
 *
 * ctx[in] -- 摘要上下文对象
 * md[out] -- 消息摘要数据
 * s[out] -- 消息摘要的数据长度
 *
 * 调用成功，返回 1，否则返回 0
 */
int EVP_DigestFinal_ex(EVP_MD_CTX *ctx, unsigned char *md, unsigned int *s);
```

我们可以将字符串 “12345678” 包括后面的 '\0' 字符一起拆分为 3 组原文数据来测试，以 SM3 算法为例，如下示例 2 所示：

``` C
#include <stdlib.h>
#include <string.h>
#include <openssl/evp.h>

#include <trace/trace.h>

int main(int argc, char *argv[])
{
    /* 定义原文数据和输出缓冲区 */
    unsigned char data1[] = "123";
    unsigned char data2[] = "456";
    unsigned char data3[] = "78";
    unsigned char *out = NULL;

    size_t mdlen = 0;
    EVP_MD_CTX *ctx = NULL;

    EVP_MD *sm3 = EVP_MD_fetch(NULL, "SM3", NULL);
    if (!sm3) {
        TRACE("获取算法失败！\n");
        return 0;
    }

    ctx = EVP_MD_CTX_new();
    if (!ctx) {
        TRACE("创建摘要上下文失败！\n");
        goto end;
    }

    /* 获取摘要长度 */
    mdlen = EVP_MD_get_size(sm3);
    TRACE("SM3 算法摘要长度为%d\n", (int)mdlen);

    /* 为摘要数据申请内存 */
    out = malloc(mdlen);
    if (!out) {
        goto end;
    }
    memset(out, 0, mdlen);

    if (EVP_DigestInit_ex2(ctx, sm3, NULL) != 1) {
        TRACE("初始化失败！\n");
        goto end;
    }

    if ((EVP_DigestUpdate(ctx, data1, strlen((const char *)data1)) != 1) ||
        (EVP_DigestUpdate(ctx, data2, strlen((const char *)data2)) != 1) ||
        (EVP_DigestUpdate(ctx, data3, sizeof(data3)) != 1)) {
        TRACE("摘要运算失败！\n");
        goto end;
    }

    if (EVP_DigestFinal_ex(ctx, out, (unsigned int *)&mdlen) != 1) {
        TRACE("获取摘要数据失败！\n");
        goto end;
    }

    TRACE_BIN("Digest", out, mdlen);

end:
    if (out) {
        free(out);
    }

    if (ctx) {
        EVP_MD_CTX_free(ctx);
    }

    if (sm3) {
        EVP_MD_free(sm3);
    }
    return 0;
}
```

运算结果如下：

``` BASH
SM3 算法摘要长度为32
Digest size:32
------------------------+------------------------
d3 70 75 73 6f 58 21 70 | 18 a9 ac ba a1 5f 7d 63
58 a9 dd 21 26 cd b5 34 | be 38 7a ef 0c 58 83 40
------------------------+------------------------
```

结果和示例1完全一致。由此可见，哪怕是将原文数据拆分，只要运算顺序不变，最后得出的结果也是一致的，这样我们在遇到大容量的原文数据时，就可以利用分组摘要接口分组处理以节省内存。

以上便是本章的全部内容，下一章节我计划和读者一起学习公开密钥算法相关的 API。
