---
title: Openssl 加密算法库编程精要 05 详解 EVP API 公开密钥密码算法 -- 生成密钥对
date: 2022-02-14 14:56:00
tags: [Openssl, Programming]
categories:
  - 信息安全
math: true
---

简要介绍 Openssl 生成公开密钥对的方式

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. 《初等数论及其应用》Kenneth H.Rosen 夏鸿刚译；
4. 《算法导论》Thomas H. Cormen、Charles E. Leiserson、Ronald L. Rivest、Clifford Stein，殷建平、徐云、王刚、刘晓光、苏明、皱恒明、王宏志译
5. 《现代密码学及其应用》Richard E.Blahut 黄玉划、薛明富、许娟译；
6. 《GB/T 32918.1-2016 信息安全技术 SM2 椭圆曲线公钥密码算法 第 1 部分：总则》
7. [知乎 《RSA 算法正确性证明》 作者：林扣星](https://zhuanlan.zhihu.com/p/48994878)
8. 《RFC3447 Public-Key Cryptography Standard (PKCS) #1: RSA Cryptography Specifications Version 2.1》

## 5.1 公开密钥系统简介

公开密钥系统最早于上世纪 70 年代被发明。在这种密码系统中，已知加密密钥，在现有计算机技术条件下很难快速求出解密密钥，这个推导过程耗费的计算机算力巨大到不切实际，所以加密密钥是可以公开的，所以这种系统被称为公开密钥系统。公开密钥系统被广泛地用于各种密码协议、数字签名以及电子商务等各种领域中。

## 5.2 RSA 算法

公开密钥系统使用的算法最流行的当属 RSA 算法，它由 Ronald Rivest、Adi Shamir 和 Lenoard Adleman 于上世纪70 年代发明，该算法的安全性基于大数分解的难度，它的原理如下:

 设：明文为 $P$ ，密文为 $C$，加密函数 $E(x)$，解密函数 $D(x)$

1. 首先选取一个公钥指数 $e$，同时生成两个大素数 $p$ 和 $q$；
2. 计算 $n = pq$, 同时计算欧拉函数 $\phi(n) = \phi(pq) = \phi(p)\phi(q) =(p - 1)(q - 1)$, 确保当前公钥指数 $e$ 和 $\phi (n)$ 互素，那么 $e$ 和 $n$ 组成的数对 $(e, n)$ 即为公钥；
3. 加密过程为 $E(P) = C\equiv P^{e}(mod\ n)$, $0\leqslant C< n$；
4. 由于 $e$ 和 $n$ 互素，所以存在一个 $e$ 的逆 $d$，使得 $ed \equiv 1(mod\ \phi(n))$ 成立，故解密时需要先解线性同余方程求出 $d$；
5. 求出 $d$ 之后，解密过程则为 $D(C) = C^{d} = ( P^{e} )^{d} = P^{ed}$；
6. 由于 $ed \equiv 1\ (mod\ \phi(n))$ 成立，所以存在一个整数 $k$，使得等式 $ed = k\phi(n) + 1$ 成立；
7. 如果 $P$ 和 $n$ 互素，由欧拉定理得出，$P^{\phi(ed)} \equiv P^{k\phi(n) + 1} \equiv PP^{k\phi(n)} \equiv P(mod\ n)$；
8. 一般情况下，$P$ 和 $n$ 不互素的概率极小，但是如果 $P$ 和 $n$ 不互素，那么由于 $P$ 必然小于 $n$ (如果 $P > n$ ，则无法通过解密算法计算出明文) ，$p$ 和 $q$ 又都是素数，那么 $P$ 必然符合以下的条件；

    - 如果 $P = ap$ ， 那么 $P$ 和 $q$ 互素
    - 如果 $P = bq$ ， 那么 $P$ 和 $p$ 互素
    - 所以我们假设 $P = ap$， 那么 $P$ 和 $q$ 根据欧拉定理得：
    
        - $ P^{\phi(q)} \equiv 1\ (mod\ q\ ) $
        - 然后有 $ (P^{\phi(q)})^{k\phi(p)} \equiv (1)^{k\phi(p)}(mod\ q\ ) $
        - 得 $ P^{k\phi(n)} \equiv 1\ (mod\ q\ ) $
        - 然后根据同余的定义可知，存在一个数 $r$，使得以下式子成立：

            - $P^{k\phi(n)} = 1 + rq$，两边同时乘以 $P$ ，得
            - $P^{k\phi(n) + 1} = P + rqP$，又因为 $P = ap$，所以
            - $P^{k\phi(n) + 1} = P + rqap$，然后由于 $n = pq$，所以
            - $P^{k\phi(n) + 1} = P + ran$，由于 $r$、$a$ 是常量，所以最终得
            - $P^{k\phi(n) + 1} \equiv  P(mod\ n)$，进一步得到
            - $P^{ed} \equiv  P(mod\ n)$

所以我们证明了使用  $d$ 可以解密出加密数据，那么数对 $(d，n)$ 即为私钥，但是原文 $P$ 必须小于模数 $n$ ，否则无法正确解密。

## 5.3 椭圆曲线密码算法

关于椭圆曲线算法，由于其数学原理较为深奥复杂，且笔者能力和精力有限，所以对于这一部分知识，我这里不做介绍，这里我推荐两本相关的资料：
1. 《现代密码学及其应用》（Richard E.Blahut 黄玉划、薛明富、许娟译），这本书里面系统地介绍了椭圆曲线密码学里面的各种概念和数学公理，不过内容确实有些深奥难懂；
2. 《信息安全技术 SM2 椭圆曲线公钥密码算法 第 1 部分：总则》，这个标准对用到的数学原理的讲解虽然并不系统，但是胜在通俗易懂，非常容易理解。

这里我简单摘录国家标准文档《信息安全技术 SM2 椭圆曲线公钥密码算法 第 1 部分：总则》的一部分内容来为各位读者介绍椭圆曲线的基本知识。
N.Koblitz 和 V.Miller 在 1985 年各自独立地提出将椭圆曲线应用于公钥密码系统，椭圆曲线公钥密码所基于的曲线性质如下：
1. 有限域上椭圆曲线在点加运算下构成有限交换群，且其阶与基域规模相近；
2. 类似于有限域乘法群中的乘幂运算，椭圆曲线多倍点运算构成一个单向函数；

在多倍点运算中,已知多倍点与基点,求解倍数的问题称为椭圆曲线离散对数问题。对于一般椭圆曲线的离散对数问题,目前只存在指数级计算复杂度的求解方法。与 RSA 算法的大数分解问题及有限域上离散对数问题相比，椭圆曲线离散对数问题的求解难度要大得多。因此,在相同安全程度要求下,椭圆曲线密码较其他公钥密码所需的密钥规模要小得多。
特别强调一下，我们整个系列博文都只会用到 RSA 和 SM2 两种公钥算法。对于其他的公钥算法比如 DSA 或者 ECDSA等算法不做讨论。

## 5.4 生成密钥对

Openssl 提供了生成公钥密码算法密钥对所需的函数，如下所示：

``` C
#include <openssl/evp.h>

/**
 * 快速生成非对称密钥对
 *
 * libctx[in] -- Openssl 库上下文
 * propq[in] -- 属性查询字符串
 * type[in] -- 算法类型，带参数
 *
 * 成功，返回密钥对象，否则返回 NULL
 */
EVP_PKEY *EVP_PKEY_Q_keygen(OSSL_LIB_CTX *libctx, const char *propq, const char *type, ...);

/**
 * 创建公钥密码算法上下文
 *
 * libctx[in] -- Openssl 库上下文
 * name[in] -- 算法名称
 * propquery[in] -- 属性查询字符串
 *
 * 成功，返回上下文对象，否则返回 NULL
 */
EVP_PKEY_CTX *EVP_PKEY_CTX_new_from_name(OSSL_LIB_CTX *libctx, const char *name, const char *propquery);

/**
 * 销毁公钥密码算法上下文
 *
 * ctx[in] -- 公钥密码算法上下文
 */
void EVP_PKEY_CTX_free(EVP_PKEY_CTX *ctx);

/**
 * 初始化公钥上下文
 *
 * ctx[in] -- 公钥密码算法上下文
 *
 * 成功，返回 1，否则返回 0
 */
int EVP_PKEY_keygen_init(EVP_PKEY_CTX *ctx);

/**
 * 生成密钥对
 *
 * ctx[in] -- 公钥密码算法上下文
 * ppkey[out] -- 密钥对象
 *
 * 成功，返回 1，否则返回 0
 */
int EVP_PKEY_generate(EVP_PKEY_CTX *ctx, EVP_PKEY **ppkey);

/**
 * 销毁密钥
 *
 * key[in] -- 密钥对象
 */
void EVP_PKEY_free(EVP_PKEY *key);
```

我们利用以上接口来实现生成 SM2 密钥对和 RSA-2048 密钥对，示例 1 如下：

``` C
#include <openssl/evp.h>
#include <trace/trace.h>

/* 我们用两种方式来生成密钥 */

/* 快速生成 SM2 密钥 */
void gen_sm2_key_pair_quickly()
{
    EVP_PKEY *pair = EVP_PKEY_Q_keygen(NULL, NULL, "EC", "SM2");
    
    if (!pair) {
        TRACE("生成 SM2 密钥失败");
        return;
    }

    /* 打印密钥 */
    EVP_PKEY_print_private_fp(stdout, pair, 0, NULL);
    EVP_PKEY_free(pair);
}

/* 手动设置密钥生成参数，生成 RSA2048 密钥 */
void gen_rsa_key_pair()
{
    EVP_PKEY *pkey = NULL;
    OSSL_PARAM params[3];
    unsigned int bits = 2048;

    /* 新版本 Openssl 默认的 RSA 公钥指数为 65537，这里为了演示刻意为之 */
    unsigned int e = 65537;

    EVP_PKEY_CTX *ctx = EVP_PKEY_CTX_new_from_name(NULL, "RSA", NULL);
    if (!ctx) {
        TRACE("生成上下文失败\n");
        return;
    }

    EVP_PKEY_keygen_init(ctx);

    params[0] = OSSL_PARAM_construct_uint("bits", &bits);
    params[1] = OSSL_PARAM_construct_uint("e", &e);
    params[2] = OSSL_PARAM_construct_end();

    if (EVP_PKEY_CTX_set_params(ctx, params) != 1) {
        TRACE("设置参数失败\n");
        goto end;
    }

    if (EVP_PKEY_generate(ctx, &pkey) != 1) {
        TRACE("生成 RSA 密钥失败\n");
        goto end;
    }

    /* 打印密钥 */
    EVP_PKEY_print_private_fp(stdout, pkey, 0, NULL);

end:
    if (pkey) {
        EVP_PKEY_free(pkey);
    }

    EVP_PKEY_CTX_free(ctx);
}

int main(int argc, char *argv[]) 
{
    gen_sm2_key_pair_quickly();
    gen_rsa_key_pair();
    return 0;
}
```

结果如下：

``` BASH
Private-Key: (256 bit)
priv:
    f8:2d:71:60:47:6f:77:bc:9c:36:4c:94:d2:66:e4:
    80:e0:1b:df:20:81:35:da:f1:4f:8f:1d:66:22:df:
    d6:5d
pub:
    04:5f:f8:b2:21:fd:97:02:0b:e8:b9:ff:a1:c8:5c:
    c0:d5:33:4a:96:84:88:ee:3c:10:5e:5c:17:11:14:
    49:93:07:0d:67:20:05:9d:c8:93:2f:79:e9:a5:0c:
    74:2b:82:30:76:83:89:3f:ec:c7:6f:ef:3c:9a:b7:
    cd:ec:c9:ac:f7
ASN1 OID: SM2
Private-Key: (2048 bit, 2 primes)
modulus:
    00:92:de:79:2e:76:04:c3:cb:99:25:fe:c8:5d:ba:
    ca:ee:24:51:43:65:14:ec:9f:c6:2a:10:87:10:4f:
    dc:9a:77:09:c8:aa:eb:cf:6b:b0:39:6b:58:f7:a5:
    e3:53:f0:e0:03:2c:d0:bb:2a:2a:b0:18:c5:30:ed:
    3f:07:b6:d4:dd:96:b2:8c:c7:e5:49:98:a9:56:de:
    14:53:61:d4:31:28:2a:d2:88:31:78:f3:59:1f:87:
    cd:19:39:6d:b5:a4:76:86:7b:5b:c5:02:03:2a:64:
    21:c9:b8:c9:6f:2b:a1:aa:f2:99:eb:83:7a:5f:54:
    6d:bb:ce:96:f1:4a:2c:74:d7:6b:da:15:84:ff:7c:
    84:9f:04:62:76:54:42:3e:01:41:13:5e:07:cd:3d:
    2c:6d:6b:d6:a6:7b:47:45:81:53:4a:f2:9e:f7:7c:
    be:e7:e6:5b:c2:09:77:5b:76:37:ab:17:ac:9a:5b:
    e9:d5:b6:18:d1:20:07:97:61:e6:50:0f:9d:74:b3:
    0d:2f:18:83:35:8d:9f:36:1e:17:7c:bf:c9:e1:83:
    9c:cf:82:ce:b2:e6:4e:ab:f9:88:fb:0e:74:c8:ae:
    66:64:46:3a:ac:c1:3f:e4:b0:0b:f1:97:ee:8c:66:
    fa:2f:d2:b3:98:1f:b8:aa:e7:eb:82:b7:67:4b:18:
    d1:79
publicExponent: 65537 (0x10001)
privateExponent:
    0d:f6:9e:86:81:cc:4d:d7:32:13:7a:dd:39:8d:69:
    aa:14:2e:db:b2:fa:61:cd:86:4d:77:d9:22:29:0d:
    9c:16:9c:34:b0:0c:ba:61:8e:80:19:b4:d1:85:66:
    8f:13:a9:75:f3:d1:51:32:1e:f1:33:71:b0:07:51:
    04:f3:e7:84:bd:1a:9d:a8:35:a4:4b:cc:dc:72:ec:
    73:74:56:1f:14:51:64:9c:73:e3:aa:c8:90:3c:5c:
    e3:67:37:dc:86:00:8a:24:4e:a3:1f:8a:b0:81:93:
    a0:20:f4:44:75:b2:a9:aa:3b:63:0e:52:d8:9f:12:
    23:17:19:a2:62:d4:70:de:93:5e:3c:fc:1a:7f:93:
    0c:e2:95:b8:ad:31:16:e7:1f:b9:a0:54:8a:91:32:
    c2:76:62:5b:d9:b4:9d:87:0d:ec:b5:95:68:3b:1d:
    59:82:54:12:e5:49:cd:20:b6:f0:33:19:9e:6a:35:
    24:aa:20:ba:70:7a:e2:00:c9:b1:7e:cf:a8:61:b4:
    f5:c2:4a:71:26:75:f8:83:0c:19:d4:6f:d8:bd:8a:
    0e:75:f0:a6:3b:a0:2e:88:a2:8d:99:1c:60:10:77:
    fa:d8:65:a7:5e:f5:e3:38:2c:a0:eb:e2:6c:f3:f6:
    2f:9a:26:26:ef:70:09:de:b2:15:42:04:e9:82:28:
    af
prime1:
    00:b6:24:70:3e:3c:6e:c1:0d:2b:f9:2e:c9:6c:5c:
    2c:8f:18:c4:3b:d9:1c:cd:66:e5:1d:af:27:6c:18:
    97:5b:d4:6b:72:79:77:f5:f5:a8:29:7d:ac:8c:30:
    16:1c:c2:7a:f5:af:f3:4f:2e:73:c2:4f:50:9e:fc:
    23:db:9f:b6:c2:7f:fe:89:8f:3b:ec:b4:d6:a0:9b:
    83:c5:2a:c3:75:c7:f2:21:72:64:9d:ea:23:6b:c5:
    22:22:45:31:c6:93:f4:23:3e:42:93:f9:7f:8a:b3:
    2c:63:b7:9b:54:c4:49:a9:6b:ba:a4:30:9a:1a:6f:
    30:17:b9:e8:7f:cf:69:0f:6f
prime2:
    00:ce:6c:6f:b8:70:d3:97:4d:1c:1a:57:af:93:72:
    f5:56:f3:0f:28:a7:99:cd:16:99:42:78:a8:23:7d:
    ac:d8:aa:41:44:9e:0b:59:3a:75:3d:c8:4f:38:d8:
    90:e0:d9:ce:51:d2:ac:c5:2e:bf:e6:b0:53:1d:f9:
    72:0e:2e:e7:cd:06:d2:a1:3d:f5:06:9b:bd:5d:60:
    71:e1:a7:5f:df:65:08:c5:0d:2d:ac:eb:20:10:e7:
    96:a7:28:6c:71:05:8f:78:2b:b7:71:8b:62:86:5e:
    57:e9:e4:46:21:61:07:0c:4b:26:83:12:5a:94:e3:
    24:07:78:f5:d1:c2:df:39:97
exponent1:
    75:48:91:5e:01:db:ef:43:64:05:58:33:2b:2b:4f:
    25:f5:74:a6:74:ef:2e:f4:0a:a4:4a:9c:bf:e6:35:
    d0:53:bf:bc:3e:ab:18:1d:ce:e2:a8:a1:ea:c3:2b:
    f9:e8:e0:f4:43:10:10:f4:80:65:a6:5c:eb:82:c0:
    34:33:6b:a3:62:77:ac:6c:26:d2:0c:c0:07:3b:1c:
    66:61:5a:eb:04:8c:cd:2c:b3:cd:5b:6e:e3:7e:54:
    b4:6c:89:d8:ac:7c:90:15:0f:19:e9:96:4e:e1:80:
    bb:d5:06:98:56:ac:78:03:7e:73:2b:38:8f:bc:f8:
    e2:ce:3a:ff:d1:b6:7c:d1
exponent2:
    12:ff:c8:08:a1:d9:d7:c3:31:22:fb:8f:1d:73:27:
    41:a9:7d:6b:b0:81:67:6e:fd:0c:31:2e:c8:95:78:
    a3:38:88:69:58:62:93:03:de:66:a1:59:29:52:45:
    83:6c:88:a0:df:53:27:92:f5:f6:b5:a3:f0:ce:54:
    c1:19:70:1c:5e:d4:64:22:df:ba:8b:fb:11:ed:1e:
    8e:36:69:8c:96:30:08:72:fe:11:3c:52:e7:3b:69:
    92:59:16:22:10:f0:f3:8e:92:83:d0:e0:70:9d:9e:
    59:d8:b8:db:b9:a2:7c:6f:2e:4c:42:14:34:3f:f3:
    c0:fc:51:23:cd:5b:de:61
coefficient:
    37:12:70:d6:e1:58:05:2d:c3:1e:43:f1:25:09:e7:
    5d:ef:0f:c2:a8:2b:ba:b7:15:7e:1a:8f:1f:86:e1:
    bb:9a:6f:14:e9:b4:9a:b1:19:5c:fa:56:0a:1e:00:
    f6:a2:35:a9:19:a6:0f:69:38:01:cf:37:9f:99:50:
    27:aa:d2:84:6f:e7:54:2f:5a:cc:04:d6:c0:75:b9:
    0e:88:6c:ce:30:19:ed:b5:b4:a9:0c:bf:4e:f9:f4:
    75:46:ae:01:b4:d6:85:6c:04:b0:c6:53:9f:00:4d:
    86:a9:ef:53:b1:a3:b0:de:44:e2:86:36:2e:66:82:
    5d:ea:9e:a0:03:85:66:91
```

由此可见，SM2 的私钥 d 是一段 32 字节（256 位）的随机数，根据标准它满足 $d \in [1, n - 2]$ （$n$ 是曲线的规模），公钥 $P$ 则是按照 $P = [d]G$ 运算得出的坐标点 $(x, y)$ 的集合，最终公钥的数据为 0x04 || x || y 总共 65 个字节，由于曲线的参数公开，我们已知 $d$ 和 $G$ 情况下可以直接算出公钥 $P$，所以 SM2 的私钥并不需要包含公钥。

RSA 则是完全按照 PKCS1 标准，私钥包含的参数较多，除了模数 modulus、公钥指数 publicExponent、私钥指数 privateExponent 以及两个素数 prime1 和 prime2，还包含了另外 3 个为了方便使用中国剩余定理加速运算的变量：

- exponent1，以 a 来代替，满足 $ea \equiv 1\ (mod\ p - 1)$
- exponent2，以 b 来代替，满足 $eb\ \equiv 1\ (mod\ q - 1)$
- coefficient，以 c 来代替，满足 $qc \equiv 1\ (mod\ p)$ ，且 $c < p$

以上我们分别演示了采用两种不同的方法来生成密钥的过程，一种是利用 EVP_PKEY_Q_keygen 接口，这种方法调用简单快捷，只需要几个参数就可以调用；另一种是利用 EVP_PKEY_generate，这种方式需要先生成公钥密码算法上下文，然后针对不同的算法设置相应的参数。生成的密钥对象我们可以利用 EVP_PKEY_print_private_fp 接口来打印密钥。

以上是本章节所有内容，下一章节计划学习非对称加解密和签名验签。
