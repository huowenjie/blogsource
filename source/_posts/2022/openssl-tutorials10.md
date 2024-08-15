---
title: Openssl 加密算法库编程精要 10 ASN.1 模块攻略，DER 编解码
date: 2022-04-13 16:44:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

自定义 ASN.1 编码类型

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. 《GB/T 35276-2017 信息安全技术 SM2 密码算法使用规范》

## 10.1 Openssl ASN.1 编解码工具

Openssl 的编解码工具主要主要包含在 asn1.h 和 asn1t.h 这两个头文件里。对于每一个 ASN.1 类型，Openssl 都会为其定义5个函数，我在这里称其为编解码工具函数，如下所示：

``` C
/* ASN1 对象的创建和销毁 */
extern TYPE *TYPE_new(void);
extern void TYPE_free(TYPE *a);

/* DER 解码 */
extern TYPE *d2i_TYPE(TYPE **a, const unsigned char **in, long len);

/* DER 编码 */
extern int i2d_TYPE(const TYPE *a, unsigned char **out);

/* 编码对象结构的引用 */
extern const ASN1_ITEM * TYPE_it(void);
```

为了方便使用，Openssl 在 asn1.h 头文件中提供了 “DECLARE_ASN1_FUNCTIONS” 宏来快速声明以上 5 个函数，同样的，Openssl 也提供了快速定义这 5 个函数的宏模板 “IMPLEMENT_ASN1_FUNCTION”，这个宏定义在 asn1t.h 头文件中，它可以处理大部分自定义 ASN.1 的数据类型。当然，对于基本 ASN.1 数据类型，比如 ASN1_OBJECT、ASN1_INTEGER 、ASN1_OCTET_STRING 等，Openssl 并不一定完全按照这套模板来定义，比如 ASN1_OBJECT 类型的编解码工具是直接定义在 crypto/asn1/a_object.c 文件中；而 ASN1_INTEGER 类型的编解码工具则是定义在 crypto/asn1/tasn_typ.c 文件中，而且它并没有直接调用 IMPLEMENT_ASN1_FUNCTION 宏。

asn1.h 头文件里声明了绝大多数的基础 ASN.1 数据类型的编解码工具函数；asn1t.h 则是定义了构建 ASN.1 编解码工具的模板。我们以 ASN1_INTEGER 类型为例，Openssl 给 ASN1_INTEGER 类型内置了一组编解码工具，它的声明如下所示：

``` C
DECLARE_ASN1_FUNCTIONS(ASN1_INTEGER)
```

为了程序的可扩展性或者某些历史原因（笔者猜测），“DECLARE_ASN1_FUNCTIONS” 宏嵌套了大量其他的宏定义，为了理清它的预处理流程，我们将 “DECLARE_ASN1_FUNCTIONS” 宏定义按顺序列举在下面：

``` C
/* 声明编解码函数 */
#define DECLARE_ASN1_FUNCTIONS(type) DECLARE_ASN1_FUNCTIONS_attr(extern, type)

/* 函数前添加属性关键字 */
#define DECLARE_ASN1_FUNCTIONS_attr(attr, type) DECLARE_ASN1_FUNCTIONS_name_attr(attr, type, type)

/* 分开声明内存管理和编解码函数 */
#define DECLARE_ASN1_FUNCTIONS_name_attr(attr, type, name) \
    DECLARE_ASN1_ALLOC_FUNCTIONS_name_attr(attr, type, name) \
    DECLARE_ASN1_ENCODE_FUNCTIONS_name_attr(attr, type, name)

/* 按照类型名称声明内存管理工具 */
#define DECLARE_ASN1_ALLOC_FUNCTIONS_name_attr(attr, type, name) \
    attr type *name##_new(void); \
    attr void name##_free(type *a);

/* 按照名称声明编解码工具 */
#define DECLARE_ASN1_ENCODE_FUNCTIONS_name_attr(attr, type, name) \
    DECLARE_ASN1_ENCODE_FUNCTIONS_attr(attr, type, name, name)

/* 分开声明 ASN1 类型对象和编解码工具函数 */
#define DECLARE_ASN1_ENCODE_FUNCTIONS_attr(attr, type, itname, name) \
    DECLARE_ASN1_ENCODE_FUNCTIONS_only_attr(attr, type, name) \
    DECLARE_ASN1_ITEM_attr(attr, itname)

/* 编解码工具 */
#define DECLARE_ASN1_ENCODE_FUNCTIONS_only_attr(attr, type, name) \
    attr type *d2i_##name(type **a, const unsigned char **in, long len); \
    attr int i2d_##name(const type *a, unsigned char **out);

/* ASN1_ITEM 对象 */
#define DECLARE_ASN1_ITEM_attr(attr, name) \
    attr const ASN1_ITEM * name##_it(void);
```

我们指定 “DECLARE_ASN1_FUNCTIONS” 的宏参数是 ASN1_INTEGER，经过一系列展开，我们最终会得到如下的函数声明：

``` C
/* ASN1 对象的创建和销毁 */
extern ASN1_INTEGER *ASN1_INTEGER_new(void);
extern void ASN1_INTEGERT_free(ASN1_INTEGER *a);

/* DER 解码 */
extern ASN1_INTEGER *d2i_ASN1_INTEGER(ASN1_INTEGER **a, const unsigned char **in, long len);

/* DER 编码 */
extern int i2d_ASN1_INTEGER(const ASN1_INTEGER *a, unsigned char **out);

/* 编码对象结构的引用 */
extern const ASN1_ITEM * ASN1_INTEGER_it(void);
```

由此可见，我们通过调用 “DECLARE_ASN1_FUNCTIONS” 宏得到了对象的编解码声明，我们前面提到，ASN1_INTEGER 编解码工具的实现并没有直接调用 “IMPLEMENT_ASN1_FUNCTION” 宏，而是调用了“IMPLEMENT_ASN1_STRING_FUNCTIONS” 宏，它定义于 Openssl 源码目录下面 crypto/asn1/tasn_typ.c（不同版本的 Openssl 可能位置不同）文件：

``` C
#define IMPLEMENT_ASN1_STRING_FUNCTIONS(sname) \
    IMPLEMENT_ASN1_TYPE(sname) \
    IMPLEMENT_ASN1_ENCODE_FUNCTIONS_fname(sname, sname, sname) \
    sname *sname##_new(void) \
    { \
        return ASN1_STRING_type_new(V_##sname); \
    } \
    void sname##_free(sname *x) \
    { \
        ASN1_STRING_free(x); \
    }
```

从这个 “IMPLEMENT_ASN1_STRING_FUNCTIONS” 宏可以看出它直接定义了一组内存管理函数，通过调用 ASN1_STRING_type_new 和 ASN1_STRING_free 来实现；同时，“IMPLEMENT_ASN1_STRING_FUNCTIONS” 宏首先通过调用 “IMPLEMENT_ASN1_TYPE” 定义了 TYPE_it 函数，“IMPLEMENT_ASN1_TYPE” 宏位于 asn1t.h 头文件中，和声明函数一样，它逐级展开后会定义一个名为 ASN1_INTEGER_it 的函数，函数返回 ASN1_ITEM *，ASN1_ITEM 是一个结构体，我们先看一下 ASN1_ITEM 的结构：

``` C
struct ASN1_ITEM_st {
    char itype;
    long utype;
    const ASN1_TEMPLATE *templates;
    long tcount;
    const void *funcs; 
    long size;
    const char *sname;
};
```

从以上 ASN1_ITEM 的结构可以看出，ASN1_ITEM 主要用于标记当前定义的 ASN.1 类型的基本信息，明确了它的功能，然后我们分析一下 “IMPLEMENT_ASN1_TYPE” 宏的展开过程：

``` C
/* type_it 函数前半部分 */
#define ASN1_ITEM_start(itname) \
    const ASN1_ITEM * itname##_it(void) \
    { \
        static const ASN1_ITEM local_it = {

/* type_it 函数后半部分 */
#define ASN1_ITEM_end(itname) \
        }; \
        return &local_it; \
    }

/* TYPE_it(void) 函数的定义 */
#define IMPLEMENT_ASN1_TYPE(stname) IMPLEMENT_ASN1_TYPE_ex(stname, stname, 0)
#define IMPLEMENT_ASN1_TYPE_ex(itname, vname, ex) \
    ASN1_ITEM_start(itname) \
        ASN1_ITYPE_PRIMITIVE, V_##vname, NULL, 0, NULL, ex, #itname \
    ASN1_ITEM_end(itname)
```

很明显，“IMPLEMENT_ASN1_TYPE” 采用了封装一个静态常量的方式来记录 TYPE 的基本信息，最终会展开成一个 TYPE_it 的函数，用户调用这个函数会得到该静态变量的地址，这个函数非常重要，因为构建大多数 ASN.1 数据类型的编解码工具时都会通过调用 TYPE_it 来获取数据的信息，从而正确的进行编解码。所以 IMPLEMENT_ASN1_TYPE(ASN1_INTEGER) 最终被展开成这样：

``` C
const ASN1_ITEM *ASN1_INTEGER_it(void)
{
    static const ASN1_ITEM local_it = {
        ASN1_ITYPE_PRIMITIVE, V_ASN1_INTEGER, NULL, 0, NULL, 0, "ASN1_INTEGER"
    };
    return &local_it;
}
```

分析完 “IMPLEMENT_ASN1_TYPE” 宏，我们再继续看 “IMPLEMENT_ASN1_ENCODE_FUNCTIONS_fname” 宏，这个宏定义在 asn1t.h 头文件中，定义如下：

``` C
# define IMPLEMENT_ASN1_ENCODE_FUNCTIONS_fname(stname, itname, fname) \
    stname *d2i_##fname(stname **a, const unsigned char **in, long len) \
    { \
            return (stname *)ASN1_item_d2i((ASN1_VALUE **)a, in, len, ASN1_ITEM_rptr(itname));\
    } \
    int i2d_##fname(const stname *a, unsigned char **out) \
    { \
            return ASN1_item_i2d((const ASN1_VALUE *)a, out, ASN1_ITEM_rptr(itname));\
    }
```

可以看到，大多数 ASN.1 类型的编解码工具都是最终调用了 ASN1_item_i2d 或者 ASN1_item_d2i 这两个函数。这两个函数在处理 ASN1 对象的时候都会将其地址转化为一个 ASN1_VALUE 的指针，ASN1_VALUE 是一个无定义的结构体，我想 Openssl 团队在设计这个类型时可能是考虑到程序的可读性、可扩展性，但它并没有什么实际意义，用户在操作 d2i 和 i2d 工具编解码时，实际上传入的依旧是用户自己创建的对象的地址。

对于 Openssl 来说，由于将 ASN1 对象地址转换为了 ASN1_VALUE *，所以 Openssl 并不知道 ASN1 对象的具体结构，所以就需要额外传入一个 ASN1_ITEM 对象，这个对象存储了用户定义的 ASN1 对象的结构信息。对于每种类型，由于我们已经通过 “IMPLEMENT_ASN1_TYPE” 定义好了 TYPE_it 函数，这个函数返回对应类型的全局 ASN1_ITEM 结构地址，所以我们可以看到 ASN1_item_d2i 和 ASN1_item_i2d 两个函数的最后一个参数都调用 Openssl 提供的宏 “ASN1_ITEM_rptr”，这个宏展开后即是函数 const ASN1_ITEM *TYPE_it(void)，这样底层就可以准确识别数据结构，正确地进行编解码操作。

那么根据以上分析，我们可以得到 IMPLEMENT_ASN1_STRING_FUNCTIONS(ASN1_INTEGER) 展开后的完整结果：

``` C
/* 定义获取 ASN.1 类型信息函数 */
const ASN1_ITEM *ASN1_INTEGER_it(void)
{
    static const ASN1_ITEM local_it = {
        ASN1_ITYPE_PRIMITIVE, V_ASN1_INTEGER, NULL, 0, NULL, 0, "ASN1_INTEGER"
    };
    return &local_it;
}

/* DER 解码 */
ASN1_INTEGER *d2i_ASN1_INTEGER(ASN1_INTEGER **a, const unsigned char **in, long len)
{
    return (ASN1_INTEGER *)ASN1_item_d2i((ASN1_VALUE **)a, in, len, ASN1_INTEGER_it());
}

/* DER 编码 */
int i2d_ASN1_INTEGER(const ASN1_INTEGER *a, unsigned char **out)
{
    return ASN1_item_i2d((const ASN1_VALUE *)a, out, ASN1_INTEGER_it());
}

/* 创建 ASN.1 对象 */
ASN1_INTEGER *ASN1_INTEGER_new(void)
{
    return ASN1_STRING_type_new(V_ASN1_INTEGER);
}

/* 销毁 ASN.1 对象 */
void ASN1_INTEGER_free(ASN1_INTEGER *x)
{
    ASN1_STRING_free(x);
}
```

## 10.2 内置基本 ASN.1 类型的 DER 编解码

我们以 ASN1_INTEGER 和 ASN1_OCTET_STRING 为例来说明对基本类型进行编解码的方式，示例代码如下：

``` C
#include <stdlib.h>
#include <string.h>

#include <openssl/asn1.h>
#include <trace/trace.h>

const char data[] = "1234567890";

/* 对象定义 */
int main(int argc, char *argv[])
{
    unsigned char *ider = NULL;
    unsigned char *bder = NULL;
    unsigned char *tmp = NULL;

    int ilen = 0;
    int blen = 0;

    ASN1_INTEGER *int_obj = ASN1_INTEGER_new();
    ASN1_OCTET_STRING *byte_obj = ASN1_OCTET_STRING_new();

    if (ASN1_INTEGER_set_int64(int_obj, 1024) != 1) {
        TRACE("设置整数失败\n");
        goto end;
    }

    if(ASN1_OCTET_STRING_set(byte_obj, (const unsigned char *)data, sizeof(data)) != 1) {
        TRACE("设置八位组数据失败\n");
        goto end;
    }

    /* 整数的 DER 编码 */
    ilen = i2d_ASN1_INTEGER(int_obj, NULL);
    if (ilen < 0) {
        TRACE("整数编码失败\n");
        goto end;
    }

    ider = malloc(ilen);
    tmp = ider;
    memset(ider, 0, ilen);

    if (i2d_ASN1_INTEGER(int_obj, &tmp) != ilen) {
        TRACE("整数编码失败\n");
        goto end;
    }

    /* 字节数据的 DER 编码 */
    blen = i2d_ASN1_OCTET_STRING(byte_obj, NULL);
    if (blen < 0) {
        TRACE("数据编码失败\n");
        goto end;
    }

    bder = malloc(blen);
    tmp = bder;
    memset(bder, 0, blen);

    if (i2d_ASN1_OCTET_STRING(byte_obj, &tmp) != blen) {
        TRACE("数据编码失败\n");
        goto end;
    }

    /* 打印编码 */
    TRACE_BIN("ASN1_INTEGER(1024)", ider, ilen);
    TRACE_BIN("ASN1_OCTET_STRING(1234567890)", bder, blen);

end:
    if (bder) {
        free(bder);
    }

    if (ider) {
        free(ider);
    }

    ASN1_OCTET_STRING_free(byte_obj);
    ASN1_INTEGER_free(int_obj);
    return 0;
}
```

结果如下所示：

``` BASH
ASN1_INTEGER(1024) size:4
-------------------------------------------------
02 02 04 00 
-------------------------------------------------
ASN1_OCTET_STRING(1234567890) size:13
-------------------------------------------------
04 0b 31 32 33 34 35 36 | 37 38 39 30 00 
-------------------------------------------------
```

内置的数据编解码比较简单，不同数据类型调用方式相对统一。基本都是先调用 TYPE_new 创建 ASN1 对象，然后将对应的数据设置给对象，然后调用 i2d_TYPE 进行编码；对于解码操作，直接调用 d2i_TYPE 即可，读者可以自己实现 DER 解码，这里仅列出编码操作。

## 10.3 复杂数据的 DER 编解码

我们这里称用户定义的 ASN1 类型本身数据类型是 SEQUENCE 的为复杂类型。和基本内置类型一样，Openssl 同样定义了很多内置的复杂类型，比如 X509、PKCS7、X509_ALGOR、PKCS12、RSAPrivateKey、RSAPublicKey、SM2_Ciphertext 等等。

我们先分析一个比较简单的 SM2 签名结构，因为 SM2 也是一种椭圆曲线算法，所以 Openssl 并没有重新定义 SM2 签名的 ASN.1 结构，而是直接采用 ECDSA 算法的 ASN.1 结构。SM2 的 ASN.1 定义如下：

``` C
SM2Signature ::= SEQUENCE {
    R INTEGER,
    S INTEGER
}
```

既然 Openssl 并没有专门为 SM2Signature 定义 ASN.1 结构，那么我们可以自己定义：

``` C
/* 定义 SM2 签名结构 */
typedef struct sm2_signature_st {
    ASN1_INTEGER *r;
    ASN1_INTEGER *s;
} SM2_SIGNATURE;

/* 声明 SM2 签名编解码工具 */
DECLARE_ASN1_FUNCTIONS(SM2_SIGNATURE)
```

我们在前面的 10.1 节就讲解了创建基本 ASN.1 类型的流程。在实现基本 ASN.1 类型的编解码工具之前，必须通过调用 IMPLEMENT_ASN1_TYPE 这个宏来创建一个全局的类型信息对象，通过 TYPE_it(void) 函数来返回这个类型信息；对于复杂的 ASN.1 类型来说，同样需要创建一个全局对象，只不过调用宏模板的方式不同，对于 SM2 SIGNATURE 结构，定义编解码工具的代码如下所示：

``` C
/* 定义 ASN.1 结构信息 */
ASN1_SEQUENCE(SM2_SIGNATURE) = {
    ASN1_SIMPLE(SM2_SIGNATURE, r, ASN1_INTEGER),
    ASN1_SIMPLE(SM2_SIGNATURE, s, ASN1_INTEGER)
} ASN1_SEQUENCE_END(SM2_SIGNATURE);

/* 定义编解码工具 */
IMPLEMENT_ASN1_FUNCTIONS(SM2_SIGNATURE)
```

从上面的示例我们可以看到，我们在自定义 ASN.1 结构时首先调用 “ASN1_SEQUENCE” 宏，这个宏的作用是声明一个静态的 ASN1_TEMPLATE 数组常量，用于描述 ASN.1 结构里面每一个成员的类型、偏移值、字段名称以及是否带有 TAG 标记等基本信息； “ASN1_SEQUENCE_END” 宏放在数组定义后面，它的功能是为编解码工具定义一个 TYPE_it（就像定义基本 ASN.1 类型时那样）；而 “ASN1_SEQUENCE” 宏则是描述结构内成员的基本信息。“ASN1_SIMPLE” 宏展开后是一个初始化 ASN1_TEMPLATE 的语句。结构里面有几个成员需要被编码，则定义几个 ASN1_SIMPLE。以我们自己定义的 SM2_SIGNATURE 为例，这些宏展开以后的形式如下：

``` C
static const ASN1_TEMPLATE SM2_SIGNATURE_seq_tt[] = {
    { 0, 0, offsetof(SM2_SIGNATURE, r),  "r", ASN1_INTEGER_it },
    { 0, 0, offsetof(SM2_SIGNATURE, s),  "s", ASN1_INTEGER_it }
};

const ASN1_ITEM *SM2_SIGNATURE_it(void)
{
    static const ASN1_ITEM local_it = {
        ASN1_ITYPE_SEQUENCE,
        V_ASN1_SEQUENCE,
        SM2_SIGNATURE_seq_tt,
        sizeof(SM2_SIGNATURE_seq_tt) / sizeof(ASN1_TEMPLATE),
        NULL,
        sizeof(SM2_SIGNATURE),
        "SM2_SIGNATURE"
    };
    return &local_it ;
}
```

以上这些示例就是创建自定义 DER 编解码工具的流程，我们可以随意找一段 SM2 签名数据来测试 DER 解码，完整代码如下所示：

``` C
#include <stdlib.h>
#include <string.h>

#include <openssl/asn1t.h>
#include <trace/trace.h>

/* 定义 SM2 签名结构 */
typedef struct sm2_signature_st {
    ASN1_INTEGER *r;
    ASN1_INTEGER *s;
} SM2_SIGNATURE;

/* 声明编解码工具 */
DECLARE_ASN1_FUNCTIONS(SM2_SIGNATURE)

int main(int argc, char *argv[])
{
    int slen = 0;
    unsigned char *data = NULL;
    const unsigned char *tmp = NULL;
    SM2_SIGNATURE *sign = NULL;

    /* 读取 sm2 签名文件 sm2-signature.sig */
    read_data("sm2-signature.sig", NULL, &slen);

    if (slen <= 0) {
        TRACE("读取数据失败\n");
        return 0;
    }

    data = malloc(slen);
    memset(data, 0, slen);

    read_data("sm2-signature.sig", data, &slen);

    /* 打印原始编码的数据 */
    TRACE_BIN("sm2-signature.sig", data, slen);

    /* DER 解码 */
    tmp = data;
    sign = d2i_SM2_SIGNATURE(NULL, &tmp, slen);
    if (!sign || !sign->r || !sign->s) {
        TRACE("解码失败\n");
        goto end;
    }

    /* 分别打印解码后 r 和 s 的数据 */
    TRACE_BIN("r", sign->r->data, sign->r->length);
    TRACE_BIN("s", sign->s->data, sign->s->length);

end:
    if (sign) {
        SM2_SIGNATURE_free(sign);
    }

    free(data);
    return 0;
}

/* 定义 ASN.1 结构信息 */
ASN1_SEQUENCE(SM2_SIGNATURE) = {
    ASN1_SIMPLE(SM2_SIGNATURE, r, ASN1_INTEGER),
    ASN1_SIMPLE(SM2_SIGNATURE, s, ASN1_INTEGER)
} ASN1_SEQUENCE_END(SM2_SIGNATURE);

/* 定义编解码工具 */
IMPLEMENT_ASN1_FUNCTIONS(SM2_SIGNATURE)
```

以上测试程序的运行结果如下：

``` BASH
sm2-signature.sig size:71
-------------------------------------------------
30 45 02 20 17 fd 8b 45 | 62 10 60 97 ce 60 71 ae
90 3c 0d 2c 64 23 4a a7 | 2e 29 ff fe ee 4f 01 6f
27 95 4d b5 02 21 00 d9 | 3f a2 eb c6 e3 7f 71 79
42 19 fd 3f cc d7 c3 8d | 11 aa af 07 f6 36 cb 17
62 25 12 ae fd 8f a7 
-------------------------------------------------
r size:32
-------------------------------------------------
17 fd 8b 45 62 10 60 97 | ce 60 71 ae 90 3c 0d 2c
64 23 4a a7 2e 29 ff fe | ee 4f 01 6f 27 95 4d b5
-------------------------------------------------
s size:32
-------------------------------------------------
d9 3f a2 eb c6 e3 7f 71 | 79 42 19 fd 3f cc d7 c3
8d 11 aa af 07 f6 36 cb | 17 62 25 12 ae fd 8f a7
-------------------------------------------------
```

以上是本章的全部内容。
