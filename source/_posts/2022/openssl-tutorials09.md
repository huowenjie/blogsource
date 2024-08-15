---
title: Openssl 加密算法库编程精要 09 ASN.1 模块攻略，Openssl ASN.1 Object
date: 2022-03-22 14:43:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

介绍 Openssl ASN.1 Object 模块的概念和用途

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. 《信息安全技术 SM2 密码算法加密签名消息语法规范》

## 9.1 OID 和 NID

OID （Object Identifier）又名对象标识符。

信息技术领域的标准化要求在全球基础上定义无歧义、可标识的标准化信息对象。这些信息对象可由不同的组织（例如政府、ISO/IEC、ITU-T 或者商务机构）定义，表示各种各样的实际对象（例如人、信息处理系统、文档、算法等）。为了满足这种需求，国际标准化组织 ISO 建立了一种信息对象注册的分层机构树，在这种结构中，“joint-iso-itu-t(2)” 和 “iso(1)” 是分层结构的第一层节点；“国家成员体”节点位于第二层“iso(1) member-body(2)”节点下；“国家”节点位于“joint-iso-itu-t(2) country(16)”节点下。我国的“国家成员体”节点和“国家”节点及其分支由国家 OID 注册中心管理。

在该分层结构下，信息对象由对象标识符（OID）唯一标识，该 OID 由从分层结构（树）的根部到叶子节点的各部分共同组成。由于从根节点到每个节点在注册机构分配的值是唯一的，故 OID 唯一。国家 OID 注册中心负责对 { iso(1) member-body(2) cn(156) } 和 { joint-iso-itu-t(2) country(16) cn(156) } 节点及其分支节点的注册、管理和维护。我们举几个通用 OID 的例子：

``` C
1.2 -- 国际标准化组织成员标识
1.2.156 -- 中国
1.2.156.10197 -- 国家密码行业标准化技术委员会
1.2.156.10197.1 -- 密码算法
1.2.156.10197.1.301 -- SM2 椭圆曲线公钥密码算法
```

Openssl 关于 OID 各个节点的定义放在 obj_mac.h 头文件中，当然这些都是提前预定义的，也是为了让 Openssl 在编译时将这些定义全部组织成一张 OBJECT 表并且内置在程序中。每一个内置 OBJECT 都会包含自己的 OID、LN_NAME（长名称）和 SN_NAME（短名称）。Openssl 内置了绝大多数关于密码相关的对象，由于数量众多，为了方便快速检索，Openssl 给每个对象都定义了一个 NID，它和 OID 一一对应，只不过是 Openssl 内部定义的，我们在程序中已知对象的 NID 即可获取 Object 对象或者其 OID；同样的，如果我们已知对象的 OID，也可以通过 OID 获取到对象的 NID。

Openssl 也允许我们自行定义 Object，假设我们要新添加一个 “SM2 密码算法加密签名消息语法规范” 的 Object，并且我们从相关标准处查得它的 OID 是 “1.2.156.10197.6.1.4.2”，处理函数包含在 openssl 头文件目录中的 objects.h 文件，如下所示：

``` C
#include <openssl3/objects.h>
#include <stdlib.h>
#include <trace/trace.h>

#define OID_SM2_CRYPTO_SIG_INFO "1.2.156.10197.6.1.4.2"
#define SN_SM2_CRYPTO_SIG_INFO "SM2 crypto sig info"
#define LN_SM2_CRYPTO_SIG_INFO "SM2 crypto and signature info"

/* 对象定义 */
int main(int argc, char *argv[])
{
    ASN1_OBJECT *obj1 = NULL;
    ASN1_OBJECT *obj2 = NULL;

    /* 创建 object */
    int nid = OBJ_create(OID_SM2_CRYPTO_SIG_INFO, SN_SM2_CRYPTO_SIG_INFO, LN_SM2_CRYPTO_SIG_INFO);
    if (nid == NID_undef) {
        TRACE("定义对象失败！\n");
        return 0;
    }

    TRACE("创建对象成功！nid = %d\n", nid);

    /* 创建 ASN1_OBJECT */
    obj1 = OBJ_nid2obj(nid);
    if (!obj1) {
        TRACE("获取对象失败!\n");
        return 0;
    }

    obj2 = OBJ_txt2obj(OID_SM2_CRYPTO_SIG_INFO, 0);
    if (!obj2) {
        TRACE("获取对象失败!\n");
        return 0;
    }

    TRACE("obj1 = %p, obj.sn = %s, obj.ln = %s\n", obj1, OBJ_nid2sn(nid), OBJ_nid2ln(nid));
    TRACE("obj2 = %p, obj.sn = %s, obj.ln = %s\n", obj2, OBJ_nid2sn(nid), OBJ_nid2ln(nid));

    return 0;
}
```

输出如下：

``` BASH
创建对象成功！nid = 1248
obj1 = 0x55b888cb7d60, obj.sn = SM2 crypto sig info, obj.ln = SM2 crypto and signature info
obj2 = 0x55b888cb7d60, obj.sn = SM2 crypto sig info, obj.ln = SM2 crypto and signature info
```

我们首先调用 OBJ_create 函数来将新的 Object 信息注册在 Openssl 内置的 Object 表中，注册成功后 Openssl 会自动分配一个 nid 作为返回值，我们可以利用这个 nid 来检索对象信息。比如调用 OBJ_nid2obj 这个函数，传入一个 nid，返回 ASN1_OBJECT 对象；调用 OBJ_nid2sn 获取对象的短名称；也可以调用 OBJ_txt2obj 函数通过 OID 字符串来检索 Object。

从示例程序输出来看，如果对象信息注册成功，不论是调用 OBJ_nid2obj 还是 OBJ_txt2obj，返回的对象地址都是一样的，所以我从这里推断 Openssl 在注册对象信息时，会将对象信息保存在全局表里自行管理，所以我们并不需要手动释放 OBJ_nid2obj 和 OBJ_txt2obj 返回的对象。

## 9.2 Openssl ASN.1 Object

说了这么多，那么我们创建 Openssl Object 到底有什么用呢？我们还是以 “PKCS7（国内类似的标准是 SM2 密码算法加密签名消息语法规范）规范” 为例来说明，先看它的通用结构定义：

``` C
ContentInfo ::= SEQUENCE {
    contentType ContentType,
    content[0] EXPLICIT ANY DEFINED BY contentType OPTIONAL
}

ContentType ::= OBJECT IDENTIFIER
```

其中，ContentType 是一个对象标识符，表示当前的内容信息包含的数据类型，content 是一个可选项，表示实际的数据内容。ASN1 解码器会根据 contentType 来选择解码不同的对象。常用的数据类型主要有 data、signedData、envelopedData、signedAndEnvelopedData、digestData 和 encryptedData 这几种。其中，PKCS7 签名数据结构 signedData 最为常用，我们看一下它的结构：

``` C
SignedData ::= SEQUENCE {
    version Version,
    digestAlgorithms DigestAlgorithmIdentifiers,
    contentInfo ContentInfo,
    certificates[0] IMPLICIT ExtendedCertificatesAndCertificates OPTIONAL,
    crls[1] IMPLICIT CertificateRevocationLists OPTIONAL,
    signerInfos SignerInfos
}
```

version 是当前数据结构的版本；digestAlgorithms 表示各个签发者所采用的摘要算法的集合；contentInfo 表示被签发的原文数据；certificates 代表证书链，用于验证签名；crls 代表证书吊销列表；signerInfos 表示签发者的列表。由此可见，我们想要构建一个 PKCS7 签名，首先要指定数据类型是签名数据类型，然后还要指定当前签发者使用的算法信息。PKCS7 签名数据类型在 Openssl 中已经有所定义，假设我们要从头实现 “SM2 密码算法加密签名消息语法规范” 签名的话，就需要首先用国密标准中指定的 OID 新创建一个对象，然后在构建 ASN.1 对象的时候依据我们新创建的签名对象，创建符合国家标准规范的签名结构。如果不针对特定的对象创建 Object，ASN.1 解码器在解码时将无法解析对象从而导致错误。

以上便是本章全部内容。
