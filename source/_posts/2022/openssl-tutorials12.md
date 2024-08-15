---
title: Openssl 加密算法库编程精要 12 PKI 体系之 X.509 证书
date: 2022-04-20 10:55:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

简要介绍 X.509 证书结构，并采用 Openssl 创建一个证书对象

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. [RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)

## 12.1 X.509 证书简介

X.509 是一个定义公钥证书格式的标准。它在 1988 年 7 月 3 日和 X.500 标准一同被发布。X.500 系统只被主权国家用于履行国家身份信息共享条约，而 IETF 的公钥基础设施（X.509）或 PKIX 工作组已经将该标准调整为更灵活的 Internet 组织。事实上，术语 X.509 证书通常指的是 IETF 的 X.509 v3 标准的 PKIX 证书和 CRL 配置文件，它在 RFC 5280 中指定，通常称为 PKIX for Public Key Infrastructure （X.509）。

X.509 证书被用于很多网络协议，比如 TLS/SSL，后者正是 HTTPS 协议的底层协议。X.509 证书同样可以用于其他的在线应用，比如电子签名。

## 12.2 X.509 证书相关的几个概念

- CRL: 证书吊销列表 （Certificate Revocation Lists），这是一种分发被 CA 认定为无效证书的信息的一种方式
- 证书路径验证算法: 证书路径算法允许中间证书签署证书，这个中间证书亦可被另外的证书签署，按着这个路径，最终到达一个信任的锚点证书（或者叫根证书）
- CSR: 证书签发请求 （Certificate Signing Request），是由申请人向公钥基础设施的注册机构（RA）发送的消息，目的是申请数字身份证书。

## 12.3 X.509 证书签发应用流程

在 X.509 系统里，如果想要签发一个证书，首先需要为用户生成一对公私钥（基于公开密钥算法），然后创建一个保存用户基本申请信息的结构 CSR 文件，这个文件包含用户的公钥、用户的申请标识、生成的证书使用的可分辨标识 DN（Distinguished Name）以及一些 CA 要求的附加信息。完成这些步骤后，使用之前生成的用户私钥对该 CSR 文件进行签名，然后将 CSR 发送给 CA。CA 根据 CSR 的信息签发证书。

## 12.4 X.509 证书结构

X.509 证书结构我们采用 ASN.1 来描述，如下所示，为了方便起见，每个结构的抬头注释我采用 C 标准多行注释：

``` C
/*
 * X.509 证书结构
 *
 * tbsCertificate -- 这个对象包含了当前主体信息、颁发者信息、与主体信息绑定的公钥等信息。
 * signatureAlgorithm -- 签名算法标识
 * signatureValue -- 证书的签名值，将 tbsCertificate 对象经过 ASN.1 DER 编码然后做摘要，最后将摘要作为原文签名的结果
 */
Certificate ::= SEQUENCE  {
    tbsCertificate TBSCertificate,
    signatureAlgorithm  AlgorithmIdentifier,
    signatureValue BIT STRING
}

/*
 * 证书主体信息
 * version -- 版本号
 * serialNumber -- 证书序列号，必定是正数，由 CA 向每个证书派发，使用证书的用户（或者应用系统）必须能够处理最多 20 个字节的序列号值，
 *     CA 派发证书序列号时不能派发超过 20 字节的值；因为历史原因，有的 CA 可能颁发序列号是负数或者 0 的证书，应用系统应针对这个问题做
 *     好兼容
 * signature -- 签名算法标识，该表示必须和 Certificate 结构的 signatureAlgorithm 表示的算法标识保持一致
 * issuer -- 颁发者，代表签署和颁发本证书的机构实体，这个字段包含了非空的可辨别名称 DN
 * validity -- 证书有效期
 * subject -- 证书主题信息
 * subjectPublicKeyInfo -- 主体公钥信息，同时表示公钥的算法
 * issuerUniqueID -- 颁发者唯一ID，使用这个字段 version 必须是 2 或者 3
 * subjectUniqueID -- 证书主题唯一ID，使用这个字段 version 必须是 2 或者 3
 * extensions -- 扩展项，使用这个字段 version 必须是 3
 */
TBSCertificate ::= SEQUENCE  {
    version [0] EXPLICIT Version DEFAULT v1,
    serialNumber CertificateSerialNumber,
    signature AlgorithmIdentifier,
    issuer Name,
    validity Validity,
    subject Name,
    subjectPublicKeyInfo SubjectPublicKeyInfo,
    issuerUniqueID [1] IMPLICIT UniqueIdentifier OPTIONAL, -- If present, version MUST be v2 or v3
    subjectUniqueID [2] IMPLICIT UniqueIdentifier OPTIONAL, -- If present, version MUST be v2 or v3
    extensions [3] EXPLICIT Extensions OPTIONAL -- If present, version MUST be v3
}

/* 版本号，一个整数，目前最新的证书版本是 V3 */
Version ::= INTEGER  {  v1(0), v2(1), v3(2)  }

/* 证书序列号，最多 20 个八位组的正整数 */
CertificateSerialNumber  ::=  INTEGER

/* 有效期 */
Validity ::= SEQUENCE {
    notBefore Time,
    notAfter Time
}

/*
 * 时间
 * utcTime -- UTC 时间，协调世界时
 * generalTime -- ASN.1 标准时间格式
 */
Time ::= CHOICE {
    utcTime UTCTime,
    generalTime GeneralizedTime 
}

/* 唯一标识 */
UniqueIdentifier ::= BIT STRING

/*
 * 和证书主题绑定的公钥信息
 * algorithm -- 算法 OID
 * subjectPublicKey -- 公钥数据
 */
SubjectPublicKeyInfo ::= SEQUENCE  {
    algorithm AlgorithmIdentifier,
    subjectPublicKey BIT STRING
}

/* 扩展项列表 */
Extensions ::= SEQUENCE SIZE (1..MAX) OF Extension

/*
 * 扩展项
 * extnID -- 扩展项 OID
 * critical -- 是否是关键项
 * extnValue -- 关键项值
 */
Extension ::= SEQUENCE  {
    extnID OBJECT IDENTIFIER,
    critical BOOLEAN DEFAULT FALSE,
    extnValue OCTET STRING
        -- contains the DER encoding of an ASN.1 value
        -- corresponding to the extension type identified
        -- by extnID
}

/* 算法定义 */
AlgorithmIdentifier ::= SEQUENCE  {
    algorithm OBJECT IDENTIFIER,
    parameters ANY DEFINED BY algorithm OPTIONAL
}
```

## 12.5 Openssl X.509 模块简介

Openssl X.509 证书结构定义在 crypto/x509.h 头文件中，而 openssl/x509.h 头文件则是为用户引出了一系列 X509 证书相关功能的接口。因为 X509 需要进行 DER 编码，所以 Openssl 为 X509 及其成员定义了 ASN.1 的信息对象和编解码工具，如下所示：

``` C
……
DECLARE_ASN1_FUNCTIONS(X509_CINF)
DECLARE_ASN1_FUNCTIONS(X509)
……
```

所以我们编程创建或者销毁 X.509 证书结构时要调用以下函数：

``` C
X509 *X509_new(void);
void X509_free(X509 *a);
```

对 X.509 进行 DER 编解码则要调用以下函数：

``` C
int i2d_X509 (const X509 *a, unsigned char **ppout);
X509 *d2i_X509(X509 **a, const unsigned char **ppin, long length);
```

为了详细了解 X.509 证书的功能和，我们从下一节开始利用 Openssl 实现一个简单的 CA 系统。以上便是本章所有内容。
