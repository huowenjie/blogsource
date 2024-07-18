---
title: Linux GCC 常用的编译、链接选项用法说明
date: 2020-09-30 15:27:00
tags: [Linux, Programming]
categories:
  - 软件开发总结
---

GCC 的命令的权威解释还是要查询[官方网站](https://gcc.gnu.org/)，同时一些链接选项不方便在网站上查询可以利用操作系统的 man 指令来查询（比如 man ld），这里记录一些常用选项，不定时更新。

<!-- more -->
## 最常用的选项

- -o file 输出目标文件
- -E 将源文件进行预处理
``` BASH
gcc -E test.c -o test.i
```

- -S 将源文件进行汇编处理
``` BASH
gcc -S test.c -o test.s
```

- -c 编译源文件
``` BASH
gcc -c test.c -o test.o
```

- 最终链接步骤
``` BASH
gcc test1.o test2.o test3.o -o test
```

- -Wall 打开所有的警告
``` BASH
gcc -c test.c -Wall -o test
```

- -O打开优化选项
- -O0 （默认）减少编译时间，生成 debug 级别的结果
- -O1/O2/O3 优化级别逐级上升，一般 release 版本的优化等级都会采用 O2 级别
``` BASH
gcc -c test.c -O2 -o test
```

- -g 生成当前系统本地格式化的调试信息， GDB 可识别并调试
- -ggdb 专门为 gdb 生成调试信息
``` BASH
gcc test.c -o test -g
gdb test
```

- -shared 生成一个可执行文件可以动态链接的共享库，这是个链接选项，编译生成共享库的目标文件的源文件时通常需要添加编译选项 -fpic
- -fpic 生成位置无关代码，在编译共享库的目标文件时使用，这是一个编译选项
``` BASH
gcc -c test.c -o test.o -fpic
gcc -shared -o libtest.so test.o
```

- -I(大写 i)  （-Idir 或者 -I dir）添加头文件搜索目录, 这是一个编译选项
``` BASH
test.c 包含 test.h, test.h 位于./inc 中
gcc -c test.c -o test.o -I inc
```

- -l(小写L) （-llib 或者 -l lib）执行链接时的共享库名称，如当前有一个共享库 libcshare.so, 那么链接命令如下：
``` BASH
gcc test.c -o test -lcshare
或
gcc test.c -o test -l cshare
```
如果当前链接目录下同时存在相同名称的共享库和静态库，比如libcshare.so、libcshare.a，在不加任何选项的情况下，编译器优先选择链接共享库，除非添加-static。

- -L （-Ldir）添加链接时共享库搜索目录；
``` BASH
gcc test.c -o test -lcshare -L/xx/xx
```
-std= 选择适配的 C/C++ 标准版本，可选的有 c89/c90/c99/c11 等；C++有c++98/c++11/c++14等等。

## 其他常用选项

### 依赖选项

- -M 为 GNU make 输出显式依赖规则，包含标准库头文件及系统头文件
- -MM 类似于 -M, 但是只会包含当前工程的头文件依赖
- -MF file 把依赖结果写入到 file
``` BASH
gcc -M test.c -Ixxx/xxx
gcc -MM test.c -Ixxx/xxx
gcc -MM test.c -Ixxx/xxx -MF test.d
```

然后查看依赖文件：

``` BASH
cat test.d
test.o: test.c xxx/xxx/test.h
```

### 常规符号导出选项

- -Wl,-Bsymbolic 优先使用本地符号, 防止链接当前共享库的应用程序中的符号覆盖当前共享库中同名的符号
- -Wl,-soname,libtest.so.1 设置共享库的 SONAME 为 libtest.so.1，readelf -d libtest.so 可以查看共享库的 SONAME
- -Wl,-rpath=/xxx/xxx 设置运行时共享库搜索目录
- -Wl,-rpath=. 设置运行时的共享库搜索目录优先选择当前目录
- -Wl,--version-script=test.map 控制共享库的导出符号

符号表的形式为:

``` BASH
VER_1 {
　　global:
　　　　testApi1;
　　　　testApi2;
　　　　testApi3;
　　local:
　　　　*;
};
 
VER_2 {
　　global:
　　　　test4;
} VER_1; #依赖于版本1
```

如果只需要控制符号表，可以写成如下形式：

``` BASH
{
　　global:
　　　　testApi1;
　　　　testApi2;
　　　　testApi3;
　　local:
     *;          
}；
```

- -Wl,--retain-symbols-file=test.sym 控制静态库的导出符号

test.sym 的格式如下:
``` BASH
testApi1
testApi2
testApi3
```

创建共享库时，添加以上链接选项 可以同时控制静态库导出符号和共享库导出符号，如下所示：
``` BASH
gcc test1.o test2.o test3.o test4.o -o libtest.so -shared -Wl,--version-script=test.map,--retain-symbols-file=test.sym
```

### 另一种控制符号导出的方式 -fvisibility

这里介绍另一种控制符号导出的方式 -fvisibility=[default|internal|hidden|protected]。

如果要公开你的接口或者 API，那么就需要将 __attribute__ ((visibility ("xxxxxx"))) 放在你需要公开的结构、类或者函数声明中，然后在编译选项中增加 -fvisibility=xxxx(可选的项有 default、internal、hidden 和 protected)。

举个例子，如果编译选项中添加 -fvisibility=hidden ，那么所有被声明为 __attribute__ ((visibility ("hidden"))) 符号都将被隐藏，其他的应用程序或者共享库在链接本库的时候会报出类似于“对 xxx 未定义的引用……”这样的错误；如果想要导出符号，则需要给符号以 __attribute__ ((visibility ("default"))) 

这样的声明，这样应用程序或者其他的共享库在链接本库时就会找到导出的符号，可以正确链接。
以下是伪码示例：

``` C++
#if __GNUC__ >= 4
    #define EXP_DEF __attribute__ ((visibility ("default")))
    #define IMP_DEF __attribute__ ((visibility ("hidden")))
#else
    #define EXP_DEF
    #define IMP_DEF
#endif
 
#ifdef __cplusplus
extern "C" {
#endif
 
// 这是 C 风格的导出函数
EXP_DEF void func(int a);
 
#ifdef __cplusplus
}
#endif
 
// C++ 导出类
class EXP_DEF person
{
public:
   person(int a);
    ……
}
```

对于其他选项，由于不是特别常用而且笔者精力有限，不做过多总结，读者可以查阅 gcc 的官方文档或者以下的参考资料来学习：

更多参考资料，见[官方文档](https://gcc.gnu.org/wiki/Visibility)

### 编译器字符集编码的控制

假设我们当前的源文件是 GBK 编码，但是我们想要让应用程序在运行时以 UTF-8 编码来显示中文，可以在编译原文件时输入以下命令：

```BASH
CFLG += -finput-charset=gbk
CFLG += -fexec-charset=utf-8gcc -c test.c -O2 -o test $(CFLG)
```

这样，程序在编译时会将源文件进行转码操作，然后运行时就会将中文字符以 UTF-8 的方式来呈现。

### 应用程序和共享库间接依赖问题

我想经常在linux下编译应用程序的朋友必定会遇到过这样的问题：  

> 在编译应用程序时，程序所依赖的共享库可能会依赖另一个共享库。  

在没有经过适当的处理的情况下，最终链接应用程序时链接器极有可能会报出找不到依赖共享库（/usr/bin/ld: warning: xxx.so, needby xxx.so, not found，try using -rpath or -rpath-link） 以及  ***未定义的引用***（/usr/bin/ld: xxx.so: undefined reference to 'xxx'）这样的错误。

现在我首先在这里复盘一下该问题，我的开发平台是 Ubuntu-Server 20.04.3 LTS-x86_64，gcc version 9.3.0。 

假设当前项目有三个目标 libalib.so、libblib.so 以及可执行程序 main，它们的源文件和头文件分别为 alib.c/alib.h、blib.c/blib.h、main.c，其中，main 依赖于 blib，blib 依赖于 alib，如下所示：

```C
/*--------------------libalib.so--------------------*/
 
/* alib.h */
#ifndef ALIB_H
#define ALIB_H
  
int afunc();
  
#endif
  
/* alib.c */
#include "alib.h"
  
int afunc()
{
    return 1024;
}
  
/*--------------------libblib.so--------------------*/
  
/* blib.h */
#ifndef BLIB_H
#define BLIB_H
  
int bfunc();
  
#endif
  
/* blib.c */
#include "alib.h"
  
int bfunc()
{
    return afunc();
}
  
/*----------------------main---------------------*/
  
/* main.c */
#include <stdio.h>
#include "blib.h"
  
int main(int argc, char *argv[])
{
    printf("b = %d\n", bfunc());
}
  
/*----------------------Makefile---------------------*/
  
CC := gcc -Wall
  
main:main.c libblib.so libalib.so
    $(CC) $< -o $@ -lblib -L.
  
libblib.so:blib.o libalib.so
    $(CC) -shared -o $@ $< -lalib -L.
  
blib.o:blib.c
    $(CC) -fPIC -c $<
  
libalib.so:alib.o
    $(CC) -shared -o $@ $< -L.
  
alib.o:alib.c
    $(CC) -fPIC -c $<
  
.PHONY:
clean:
    rm -f *.o main *.so
  
/*---------------------------------------------------*/
```

编写完程序后，输入make进行编译，结果如下:

``` BASH
gcc -Wall -fPIC -c blib.c
gcc -Wall -fPIC -c alib.c
gcc -Wall -shared -o libalib.so alib.o -L.
gcc -Wall -shared -o libblib.so blib.o -lalib -L.
gcc -Wall main.c -o main -lblib -L.
/usr/bin/ld: warning: libalib.so, needed by ./libblib.so, not found (try using -rpath or -rpath-link)
/usr/bin/ld: ./libblib.so: undefined reference to `afunc'
collect2: error: ld returned 1 exit status
make: *** [Makefile:4: main] Error 1
```

为什么会出现这个错误呢？

我的理解是，链接器在链接可执行程序的时候会进行一次运行时（RUNTIME）查找，并主动按照顺序检查所依赖共享库中的符号，如果它依赖的共享库同时依赖了其他的模块，那么它会沿着这样的依赖路径一直查找下去。如果发现有符号未定义，链接器就会报错终止，所以对于我们的例子，main 调用了 bfunc，bfunc 中调用了 afunc，那么在链接生成 main 的时候，链接器必须知道 afunc 是否定义，而我们在成功生成 libalib.so 和 libblib.so 后，由于并未设定程序运行时查找路径，所以 libalib.so 找不到 libblib.so，所以 main 无法确定 afunc 是否定义，所以有了以上报错。

那么如何解决该问题？方法有这么几种：

1. 最简单的方法就是先设置环境变量，指定运行时库目录位置，直接执行 export LD_LIBRARY_PATH=xxx，然后再进行链接；
2. 将运行时的共享库搜索目录 /etc/ld.so.conf.d/ 写入一个配置文件 xxx.conf，在这个文件里指定要搜索的目录，然后执行 ldconfig，这样会将你指定的目录记录到系统全局的目录表里，任何一个共享库都会搜索你指定的目录；
3. 链接选项添加 -Wl,-unresolved-symbols=ignore-in-shared-libs 忽略该错误，这样可以正常生成可执行程序，但是程序运行阶段依旧要解决找不到共享库的问题；
4. 采用提示的 -Wl,-rpath=xxx 指定运行时查找路径，链接器会将该路径写到应用程序，在运行时会优先查找设定的路径；
5. 采用提示的 -Wl,-rpath-link=xxx 指定运行时查找路径，链接器仅在链接阶段进行查找，并不会将该路径记录到应用程序；
6. 如果依赖的共享库数量较少，可以让应用程序依赖所有的直接、间接的共享库。
