---
title: Openssl 加密算法库编程精要 13 建立自定义 CA 01 准备工作
date: 2022-04-20 16:02:00
tags: [Openssl, Programming]
categories:
  - 信息安全
---

简述正式建立 CA 前的准备工作

<!-- more -->
## 参考资料

1. [Openssl 官方网站](https://www.openssl.org/)
2. [Openssl 官方文档](https://docs.openssl.org/master/)
3. [《Linux 内核代码风格》](https://www.kernel.org/doc/html/v4.15/translations/zh_CN/coding-style.html)

## 13.1 设计规划

我给这个程序命名为 SimpleCA，尽管它是一个用于演示学习的程序，但是我还是希望整个系统尽可能的完善、详尽。不同于我之前章节列举的那些示例程序，本章节我会依次按照我们的需求，建立一个大概的设计框架，然后按设计方案逐个实现并测试。

首先明确 SimpleCA 采用纯 C 语言开发。为了最大程度保证程序的兼容性和跨平台性，我保守地采用 C89 标准来开发这个系统，因为各个主流的 C/C++ 编译器都很好地支持了 C89 标准。同时我开发的程序本身并不涉及系统调用、网络和多线程操作（依赖的 Openssl 库会有相应的操作，但 Openssl 源代码并不在我们讨论之列，我们仅仅使用它的 API），所有的功能全部采用 C 标准库实现；同时，由于全部采用标准库开发，我并不打算专门为它设计内存管理的模块，而是直接采用标准库 stdlib.h 提供的 malloc/free 函数。

SimpleCA 最终会编译为一个类似于命令行工具的程序，用户可以输入不同的命令来执行不同的功能，它大致分为以下几个模块：

1. 错误处理，为各个模块规划错误码，可以通过错误码快速检索错误信息
2. 日志输出，提供打印日志的功能，日志可以输出到标准输出或者文件
3. 抽象数据结构，为其他模块的实现提供合适数据结构的解决方案
4. 命令管理，实现命令行控制程序的功能
5. 密钥管理，提供生成密钥、编解码密钥等功能
6. 证书管理，编解码、生成和签发证书请求以及证书的功能

以上便是 SimpleCA 系统的所有模块，在以后的实现过程中我会灵活地优化调整。

## 13.2 工程目录结构

SimpleCA 系统的目录结构如下：

``` BASH
SimpleCA
    |--inc
    |   |-xx01.h
    |   |-xx02.h
    |   |-xx03.h
    |   |-.....
    |
    |--lib
    |   |-libxxx01.so
    |   |-libxxx02.so
    |   |-libxxx03.so
    |   |-....
    |
    |--src
        |-Makefile
        |-xxx01.c
        |-xxx02.c
        |-xxx03.c
         |-...
```

## 13.3 代码风格

代码风格因人而异，我这里简单描述一下我为 SimpleCA 规划的风格。

1. 代码的长度限制

每行代码的长度尽量限制在 80 个字符

2. 缩进

所有的常规缩进都采用 4 个空格，switch 的 case 关键字和其本身对齐，匹配逻辑和 break/return/goto 等关键字需要缩进 4 个空格，同时每个 break/return/goto 后必须空一行以示分隔。对于 if、for 等语法块，保持常规缩进即可，如下所示：

``` C
switch (cond) {
case 1:
    xxxx;
    break;

case 2:
case 3:
    xxxx;
    break;
.....

default:
    break;
}

if (cond) {
    xxxx;
}

for (xxx; xxx; xxx) {
    xxxx;
}
```

3. 绝不鼓励为了不用 goto 而不用 goto 这种可笑的 “政治正确”！

存在即合理，C 语言的 goto 语句可以很方便的实现指令跳转，虽然随意乱用 goto 确实会造成一定的逻辑混乱和 bug，但是 goto 在资源释放和边界处理这些场景还是很方便的，比如：

``` C
int test()
{
    int ret = -1;
    char *str = malloc(256);
    ....
    if (err_cond) {
        print(xxx);
        goto end;
    }
    ....
    ret = 0; /* 表示成功 */
end:
    if (str) {
        free(str);
    }
    return ret ;
}
```

4. 大括号的位置

除了函数，所有大括号的起始位置都放在行尾；同时，除了像 do...while 或者 else if 等语句在大括号后有结束的关键字，否则大括号都独占一行：

``` C
if (cond) {
    xxxx;
}

void test()
{
    if (cond) {
        xxxx;
    }
}

struct type {
    int a;
    int b;
    long c;
};

do {
    ....;
} while (cond);

if (cond1) {
    xxxx1;
} else if (cond2) {
    xxxx2;
} else {
    xxxx3;
}
```

5. 所有的 for、if 、while 等关键字构成的语法块都要加大括号，哪怕逻辑只有一行：

``` C
int a;

/* 不能采用这种方式 */
if (cond)
    a = 0;

/* 应采用这种方式 */
if (cond) {
    a = 100;
}
```

6. if、switch、for、while 等关键字后面放空格，对于 if、while、for 在小括号结尾，大括号之前要有空格，如图所示：

``` C
if (cond) {
    xxxx;
}

while (cond) {
    xxxx;
}
```

7. for 语句的分号后都要加空格：

``` C
int i = 0;

for (; i < 10; i++) {
    printf("%d\n", i);
}
```

8. 函数调用时如果有多个参数，在每个逗号后都需要加空格：

``` C
const char *str = "Hello World";
char buff[32] = { 0 };

strcpy(buff, str);
printf("%s\n", buff);
```

9. 大多数二元、三元运算符两侧都要加空格，比如 ‘=’、‘<’、‘>’、‘|’、‘^’ 、‘==’、‘!=’、‘&&’、‘&’（与）、‘*’（乘） 等；

``` C
unsigned int a = 0x00000001;
unsigned int b = 0x00000002;
unsigned int c = a | b;

printf("sum = 0x%08X\n", c);
```

10. 一元操作符后面不能加空格，如 '&'（取地址）、'*'（取值）、‘+’（正）、‘-’（负）、‘sizeof’ 等；

``` C
int a = 0;
int *pa = &a;

printf("%d address is %p, and its size = %d\n", *pa, pa, (int)sizeof(a));
```

11. 自增和自减符号和变量之间都不加空格，如 “++a”，“b--”；

``` C
int i = 0;

while (i++ < 10) {
    printf("%d\n", i);
}
```

12. 合理的排版

有时候为了排版美观需要补空格对齐，我支持这种对齐排版，但是有时候如果名称长度差异过大，我们应该考虑重新优化变量或函数的命名，而不是无脑使用大量空格来强行对齐：

``` C
/* 以下的补空格对齐是完全可行的，使代码更加美观整齐，阅读也更舒服一些 */
#define CA_ALGO 0
#define CA_KEY  1
#define CA_CONT 2
....

int ca  = 0;
int key = 0;
int num = 0;
```

总而言之，美观是手段不是目的，一切都要以程序的可读性和可维护性为宗旨。

13. 注释风格

注释非常重要！因为你不知道什么时候会返回来看你的代码，如果没有注释，我相信你的脑袋准会爆炸！有一个良好的注释风格无论是对程序的可读性、可维护性甚至是美观性都是有好处的。我们的 SimpleCA 项目也不例外，现在我大概举例说明一下我的注释风格，这种风格也会用于 SimpleCA 工程。由于我采用 C89 标准，C89 是不支持 C++ 风格的 "//" 注释的，这点要格外注意。

首先我们尝试单行注释一个说明性质的文本，这个是最常用的，如下所示：

``` C
int a = 1;
int b = 2;
int t = 0;

/* 交换两个数 a 和 b （这是单行注释）*/
t = a;
a = b;
b = t;
```

这种简短的注释是很有用的， 有时候一句话就可以让你理解程序的核心或者回忆起当时开发时遇到的问题，当然做这种短注释一般要求言简意赅，尽可能说清楚你做了什么。当然不要去逐语句的描述你的程序是如何执行的，这样显得很傻而且很不专业。有时候受限于语言表达能力或者因为业务比较重要需要详细说明程序的含义，这时你就需要用更多文本说明问题，就需要用到多行注释，多行注释采用以下的风格：

``` C
/*
 * 多行注释
 * 交换两个数 a 和 b
 *
 * 这个实现耗费了我数天精力，主要是因为某某项目上急需使用，然后解决了
 *  xxxx 事情，最后的结果皆大欢喜。后面的开发者如果要接手，可以从 xxx 
 * 位置看起，因为 xxx 位置的代码有一个 xxx 的隐患，一定要注意！
 * 作者：xxx 日期 xxxx
 */
void test(int *a, int *b)
{
    int t  = 0;

    t = *a;
    *a = *b;
    *b = t;
}
```

在构建 C 工程时不可避免的要引出一些公共函数，如果这些包含公共函数的头文件要提供给他人，我们就需要对其进行 API 注释，API 注释全部采用 /** 开头，如下所示：

``` C
/**
 * 生成 RSA 密钥，以 DER 编码输出
 * 
 * 参数：
 *    bit[in] -- 模的位长
 *    der[out] -- 密钥 DER 编码数据
 *    len[in/out] -- 编码长度
 *
 * 返回值：
 *    如果成功，返回 0；如果失败，返回错误码。
 *
 * 特殊说明：
 *    如果 der 传 NULL，结果长度会赋予 len 所指向的值。
 *
 * 示例：
 *    unsigned char *der = NULL;
 *    int len = 0;
 *
 *    ca_generate_rsa_key(1024, der, &len);
 *    .....
 */
extern int ca_generate_rsa_key(int bit, unsigned char *der, int *len);
```

如果要用一些自动生成文档的软件比如 Doxygen，注释风格就应做相应的调整，这个并不统一，按实际情况来。

14. 代码逻辑分隔

一个函数中的所有业务代码不能堆在一起，应该按照逻辑功能用空格将其分开：

``` C
char global_buffer[256] = { 0 };

void set_buffer(const char *str)
{
    size_t len = 0;

    if (!str || !*str) {
        printf("Null Pointer Or Empty String!\n");
        return -1;
    }

    len = strlen(str);
    if (len > sizeof(global_buffer) - 1) {
        printf("Len too Long!\n");
        return -1;
    }

    strcpy(global_buffer, str );
    return 0;
}
```
这样除了使代码更有层次感，也可以使逻辑更加分明。

15. 如果可以的话，所有在函数入口处的变量都第一时间初始化：

``` C
void show()
{
    int a = 0;
    int b = 0;
    int c = 1;

    const char *tmp = "123";
    char buf[32] = { 0 };

    .....
}
```

16. 如果程序框架不成体系则尽量采用 C 标准库提供的默认的数据类型

不过有时候在其他嵌入式平台或者项目中如果存在大量的位操作、文件序列化和二进制数据操作，此时就特别需要以位为单位明确变量的大小，因为如 int、unsigned int 这些类型在 32 位 CPU 平台上是 32位的，而在 16 位平台的 CPU 则可能是 16 位，此时如果要直接使用 32 位整数就只能用 long 类型，而 C89 标准又不支持 stdint.h 中的 uint8_t、uint16_t 等类型，在这样的情况下，我们就需要自定义数据类型：

``` C
typedef char INT8;
typedef unsigned char UINT8;
typedef short INT16;
typedef unsigned short UINT16;
#if 32 位平台
typedef int INT32;
typedef unsigned int UINT32;
#elif 16 位平台
typedef long INT32;
typedef unsigned long UINT32;
#else
....
#endif
.....
```

或者对于一些隐藏变量或结构，我们并不知道它是什么类型，就直接这样定义：

``` C
----xxx.h----

typedef void *LINK_OBJECT;

/* 隐藏了 struct link 的结构 */
LINK_OBJECT create_link();

----xxx.h----

----xxx.c----

struct link {
    xxxx;
};

LINK_OBJECT create_link()
{
    struct link *obj = malloc(sizeof(struct link));
    ....
    return (LINK_OBJECT)obj;
}

----xxx.c----
```

17. 宏的使用

尽量别去写影响控制流程的宏，除非你清楚它在干什么！

``` C
/* 这样完全没有必要，而且新接手的开发者乍一看很难理解 */
#define TEST(x) \
    do { \
        if ((x) < 0) { \
            return -1; \
        } \
    } while (0)
```

定义有运算表达式的宏尽量用括号括起来，而且如果宏带了参数，也尽量用括号括起来，比如：

``` C
#define SUM(a, b) ((a) + (b))
```

18. 代码拆分的技巧

写程序的时候，每一句尽量在一行内完成，但是有时候单行代码不可避免变得很长，为了阅读方便需要拆分成多行。这里有几种情况，我们逐个讨论。
首先看带有表达式的长代码换行风格：

``` C
int tmp = 0;
int i = 0;
int j = 0;
int k = 0;
int x = 0;
int y = 0;

/* 这里就需要以 + 号作为分隔标记 */
tmp = pubkey_len * prvkey_len + rsa_bitlen * calc_rsa_args(&i, &j, &k) + get_pointer_coordinates(group, &x, &y);
```

先按照 “=” 做一次换行，然后按照最低优先级的 “+” 号来逐行分隔：

``` C
tmp =
    pubkey_len * prvkey_len + 
    rsa_bitlen * calc_rsa_args(&i, &j, &k) + 
    get_pointer_coordinates(group, &x, &y);
```

对于条件判断语句，也是尽量按照低优先级的逻辑运算符来分隔：


``` C
if (((SCA_UINT8 *)ta->p)[0] == 0x00 ||
    ((SCA_UINT8 *)tb->p)[0] == 0x00 ||
    ((SCA_UINT8 *)tc->p)[0] == 0x00) {
    continue;
}
```

虽然很讨厌有带有很多参数的函数，但是你不得不经常面对它，对于多参的函数，需要分隔换行的话，我一般这样来处理：

``` C
int manay_args_func(
    int x,
    int y,
    int *px,
    int *py,
    const char *name,
    const char *address,
    char *out
);
```

先将第一个参数换行，后面的参数依次换行，右括号另起一行。有的人喜欢把右括号放在最后一个参数右边，这完全没问题。我这样做是想和初始化数组或结构体的换行分隔方式统一起来：

``` C
unsigned int global_array[] = {
    0x000C0002, 0x00010002, 0x04000002, 0x0EE00002, 
    0x0AB00002, 0x0FFF0002, 0x00D00002, 0x00F00002
};

struct point3d {
    int x;
    int y;
    int z;
} points[] = {
    { 0, 0, 0 },
    { 0, 1, 0 },
    { 0, 0, 1 },
    { 1, 0, 0 },
    { 0, 1, 1 },
    { 1, 0, 0 },
    { 1, 0, 1 },
    { 0, 1, 0 },
};
```

19. 指针 * 号的位置

指针变量的 * 号靠近右边的变量名：

``` C
int *p = NULL;

typedef void *HANDLE;

const char *get_string();
```

还有许多细节方面的我这里不再赘述，总之，编码风格因人而异且无所谓对错，统一的编码风格有助于开发者维护、阅读代码，减少程序出现 bug 的几率。

## 13.4 命名规范

一个工程有一个统一的命名风格是很重要的，这点是毋庸置疑的。

1. 定义公共函数、变量、宏的时候需要统一名称空间，SimpleCA 的名称空间是 SCA：
``` C
#define SCA_UINT8 unsigned char
#define SCA_INT8 char

int sca_init();
int sca_quit();
```

2. 常量以及自定义类型全部采用纯大写字母命名，变量、函数名等均使用小写字母命名：
``` C
#define PI 3.14159265359

float radian(float angle)
{
    return angle * PI / 180.0f;
}
```

3. 不论是大写字母还是小写字母，变量名称都是使用下划线分隔；
``` C
int rsa_bit = 1024;
int ec_flag = 0;

static int global_ref_count = 0;
```

4. 头文件和源文件全部采用小写字母 + 下划线方式命名；
``` C
sca_log.h/sca_log.c
sca_test.h/sca_test.c
sca.h/sca.c
```

5. 头文件的保卫宏全部命名为 XXX_H，XXX是头文件的名称；
``` C
/* sca.h */

#ifndef SCA_H
#define SCA_H 
 ...
#endif /* SCA_H */
```

6. 局部变量命名尽量简洁，全局变量和函数命名尽量见名知意：
``` C
static int global_ref_count = 0;

int sca_init()
{
    if (global_ref_count++ > 0) {
        return 0;
    }
    ...
    return 0;
}

void sca_test()
{
    int i = 0;
    int tmp = 0;
    int *pt = NULL;
   
    ....
}
```

命名是一门大学问，如何让变量名简短但是容易让人看懂它的含义。。。没有比这更难的问题了吧。

## 13.5 编译器、编辑器、开发平台的选择

我决定采用 Ubuntu20.04-Server 来作为开发平台，VSCode 作为编辑器，gcc 作为编译器。VSCode 有一个远程连接 linux 的功能，使用 GDB 调试也是十分方便的，可以直接在图形界面上操作，这点甚至有点像 VS 了。代码、以及文件全部采用 UTF-8 编码。

## 13.6 版本管理工具

采用 Git 作为版本管理工具，代码托管在github上，项目地址是 https://github.com/huowenjie/SimpleCA。
从下一章开始，我将正式开始开发 SimpleCA，拭目以待!
