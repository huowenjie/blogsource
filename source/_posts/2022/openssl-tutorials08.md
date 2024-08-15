---
title: Openssl 加密算法库编程精要 08 ASN.1 模块攻略，ASN.1 记法语法规则
date: 2022-03-11 17:48:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

介绍 ASN.1 记法的基本语法规则和基本数据类型

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. 《ASN.1编码规则详解（最全最经典）》 -- 无名氏译
4. 《ASN.1  Communication between Heterogeneous Systems》-- Olivier Dubuisson
5. 《ASN.1 Complete》 -- J.Larmouth
6. [张明峰的博客](https://www.mingfer.cn/2019/04/01/X690-ASN1%E8%AF%AD%E6%B3%95/)

## 8.1 ASN.1 ASN.1 语法规则

一般我们定义一个 ASN.1 的类型会采用如下的格式：

``` C
<类型名称> ::= <类型>
```

类型名称在定义时需要以大写字母开头，而类型一般是内建类型或者以内建类型为基础定义的其他类型，我们举几个例子：

``` C
Age ::= INTEGER
Name ::= BIT STRING

StudentInfo ::= SEQUENCE
{
    name Name, -- 名字
    age Age
}
```

我们首先用内建类型 INTEGER 和 BIT STRING 两个类型定义了年龄（Age）、名字（Name），然后又基于这两个类型定义了一个结构 StudentInfo。ASN.1 定义的内建类型如下所示：

![](/image/2022/openssl-tutorials/OpensslAsn1Tab1.png)
![](/image/2022/openssl-tutorials/OpensslAsn1Tab2.png)

可以看出，ASN.1 为每一个类型都定义了默认的标识值，这些值又被称为 Tag 值，用于对 ASN.1 数据进行编解码。

## 8.2 BER （Basic Encoding Rules）编码

我们在传输数据时，为了使接收方可以明确区分数据类型，所以会对 ASN.1 的各种类型进行系统性的编码，一般我们采用 BER 编码。

BER 编码是 ASN.1 中最早定义的编码规则。BER 编码的传输语法格式基于 TLV 编码（<Type, Length, Value>），即类型长度值编码。DER 编码则是 BER 编码的变体，它在 BER 编码规则的基础上对许多可选项做了限制，是公钥基础设施中使用最多的编码方式。通常 BER 编码数据由以下四部分构成：

![](/image/2022/openssl-tutorials/OpensslAsn1Type.png)

1. Type 域又称为 Tag，它是类型的标识符
2. Length 域是编码数据 Value 的字节数
3. Value 域是实际数据的编码值，Value 域在 NULL 类型的时候可以省略
4. EOC 域只在编码为不定长数据的时候才会使用，用于标记数据内容的结束

为了对数据类型进行区分，ASN.1 会对每个数据类型都选择一个唯一的 Type 进行标记，数据的类型可以是 ASN.1 的内置数据类型，也可以是复合类型。
ASN.1 将数据类型编码为一个或者多个字节，编码的内容包含标记的类型 Tag class，基本数据类型还是符合数据类型 P/C，标识值 Tag number，如下所示：

![](/image/2022/openssl-tutorials/OpensslAsn1Tag.png)

在数据元素类型编码字节序列的第一个字节 Octet1 的前面两个 2 进制位，用于标识 Tag Class，Tag Class 主要有以下 4 种：

1. Universal -- 原生数据类型，Tag class 标识值为 0
2. Application -- 为特定应用设定的数据类型，Tag class 标识值为 1，不推荐使用
3. Context-specific -- 根据上下文定义的类型，Tag class 为 3
4. Private -- 私人规范中定义的类型，Tag class 为 4

第 6 个二进制位用于指定数据类型是基本数据类型(不可拆分)还是复合数据类型(可拆分)：

![](/image/2022/openssl-tutorials/OpensslAsn1Value.png)

如果定义的数据类型不是 Universal 的数据类型，那么此时需要用到更多的字节序列如 Octet2。在使用这类标记的时候，要将 Octet1 的第 5 到第 1 个二进制位置为 1 ，如果 Octet2 后面还有 Octet3，那么 Octet2 的第 8 个二进制位应该为 1。下面举例子说明一下上述编码规则：

- 编码一个 INTEGER 类型的数据
    - Tag Class 为 Universal ，对应的值 0
    - INTEGER 为 Primitive 类型，对应的值 0
    - INTEGER 的 tag number 为 2
    - 编码后的二进制位为 0000 0010，得到的 Type 域字节序列为 0x02

- 编码一个 SEQUENCE 类型的数据
    - Tag Class 为 Universal ，对应的值 0
    - SEQUENCE 为 Constructed 类型，对应的值 1
    - SEQUENCE 的 tag number 为 16
    - 编码后的二进制位为 0011 0000，得到的 Type 域字节序列为 0x30

- 编码一个自定义的基于上下文的数据类型
    - Tag Class 为 Context-specific ，对应的值 3
    - 假设该数据类型为 Primitive 类型，对应的值 0
    - 假设该数据类型的 tag number 为 300
    - 编码后的二进制位为 1101 1111 1001 0010 0000 1100，得到的 Type 域字节序列为 0xDF 0x92 0x0C

长度域的编码分为定长的长度域和不定长的长度域，其中定长的长度域分为长度不超过 127 的短格式和长度超过 127 的长格式。

![](/image/2022/openssl-tutorials/OpensslAsn1Len.png)

- 定长短格式：
    - 第一个二进制位为 0，后面的 7 个二进制位表示长度，范围为 0 - 127
    - 如 126 长度的数据，长度域二进制为 0111 1110 字节序列为 0x7E

- 定长长格式：
    - 第一个字节的第一个二进制位为 1，后面七个二级制为表示长度值所占用的字节数，后面的字节表示长度值
    - 如长度为 300 的字节长度二进制序列为 1000 0010 0000 0001 0010 1100 字节序列为 0x82 0x01 0x2C

- 不定长格式：
    - 长度域直接固定为 0x80
    - 数据内容由两个 0x00 组成的 EOC 域结束

- 保留格式：
    - 整个字节的二级制位全为 1 即 0xFF 作为保留格式

对于 Value 域，则直接保存数据的字节编码，如果 ASN.1 对象不存在或是虚对象的时候，可能没有该域，如 ASN.1 的 NULL 对象。

## 8.3 Tag 的用法

从 8.2 节我们知道了 Tag 一共有四种类型：universal，application，context-specific 和 private。那么 Tag 到底有什么用呢？我们先举一个例子：

``` C
Test ::= SEQUENCE {
    val1 INTEGER OPTIONAL,
    val2 INTEGER OPTIONAL
}
```

我们定义了一个结构体 Test，包含两个成员 val1 和 val2，OPTIONAL 表示可选。如果我们不明确指定一个变量来区分 val1 和 val2 的话，假设这个结构体仅包含 val1 或者 val2，解码器就分不清楚当前结构体包含的到底是 val1 还是 val2，最终导致歧义。

那么如何指定这个变量呢？这时就需要用到 Tag 了，Tag 主要有以下几种模式：

1. EXPLICIT tagging，编码器会默认采用这种模式，如果我们对某个成员指定了 Tag，编码器在编码的时候将会把我们指定的 Tag 值添加到目标类型（或者 tag，也可以说是 TLV 组）之前；
2. IMPLICIT tagging，采用这种模式的时候，编码器会直接将原始的 Tag 换成我们指定的 Tag 值；
3. AUTOMATIC tagging，采用这种模式的时候，对于模块内所有SEQUENCE、SET 和 CHOICE 类型， ASN.1 编译器会自动从 0 开始，步长为 1 进行自动编码。而其中的成员则用 IMPLICIT 模式，除非它是 CHOICE 类型、开放类型或者一个参数类型。

我们可以直接在成员后面添加 \[index\] 来为成员指定 Tag （index 需从 0 开始），如下所示：

``` C
Test ::= SEQUENCE {
    val1 INTEGER OPTIONAL,
    val2 INTEGER OPTIONAL
}
```

我们指定 val1 的 Tag 是 0，用来区分 val1 和 val2，这样的定义是没有歧义的，解码器也可以顺利解码。

总而言之，使用 Tag 的主要目的是避免编码时的歧义，使解码器能顺利解码。明确了这一点，我们在理解 Tag 和 ASN.1 编解码的时候就会更加容易。以上便是本章全部的内容，均是浅尝辄止，主要目的是为后面几章使用 Openssl ASN.1 模块做一个简单的铺垫。
