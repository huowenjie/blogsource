---
title: MDK 仿真环境下 STM32 RAM 调试方法
date: 2025-07-17 21:03:00
tags: [Stm32, Programming]
categories:
  - 嵌入式开发
---

本文主要记录 STM32F103C8T6 芯片在 MDK 仿真环境下采用内部 SRAM 启动程序并调试的方法。

<!-- more -->
## SRAM 调试的特点

1. 速度块，将代码下载到 SRAM 的速度远超 FLASH；
2. 理论无限次读写，增加芯片的使用寿命，在大量小批量修改代码并调试的场景，SRAM 调试方式可以显著增加调试效率，避免频繁擦写 FLASH；
3. 掉电会丢失数据；
4. 空间较小。

## 配置 STM32 SRAM 调试的步骤

### BOOT 管脚的配置

STM32 CPU 从地址 0x00000000 获取堆栈顶的地址，并从启动存储器的 0x00000004 指示的地址开始执行代码。当 BOOT 模式配置为从 FLASH 存储器启动时，这两个地址分别被映射到 0x08000000 和 0x08000004。当 BOOT 模式配置为从内置 SRAM 启动时，这两个地址分别被映射到 0x20000000 和 0x20000004。BOOT 模式的配置如下：

| BOOT1引脚 | BOOT0引脚 | 启动模式    | 映射地址 |
| -------- | --------  | ----------- | ------- |
| 任意值    | 0         | 主闪存存储器 | 0x08000000 |
| 0        | 1         | 系统存储器   | 0x1FFFB000（互联型产品），0x1FFFF000（其他产品） |
| 1        | 1         | 内置SRAM     | 0x20000000 |

基于以上信息，若要使程序从 SRAM 启动，则需将芯片的 BOOT0 管脚和 BOOT1 管脚均接高电平。

### 在原有工程的基础上增加一个 DEBUG 配置版本

![](/image/2025/stm32-sram-debug/new-debug01.jpg) ![](/image/2025/stm32-sram-debug/new-debug02.jpg) ![](/image/2025/stm32-sram-debug/new-debug03.jpg)

这样修改配置后不会影响原有项目的配置。

### 配置中断向量表

打开工程配置窗口，在C/C++一栏指定默认的宏 VECT_TAB_SRAM 即可。

![](/image/2025/stm32-sram-debug/new-debug04.jpg)

### 配置分散加载文件

这步无需手动配置，打开工程配置界面在 Target 一栏中即可配置，同时还需要在 Linker 一栏中勾选 Use Memory Layout from Target Dialog 项目，用于分配只读存储区域和可读写数据区域。

![](/image/2025/stm32-sram-debug/new-debug05.jpg) ![](/image/2025/stm32-sram-debug/new-debug11.jpg)

### 配置调试下载初始化文件

打开工程配置窗口，切换到 Debug 一栏，取消勾选 Load Application at Startup 选项同时选择初始化文件。

![](/image/2025/stm32-sram-debug/new-debug06.jpg)

初始化文件 stm32sram.ini 需要手动创建，该配置文件的内容为：

``` C
/***********************************************************/
/* Debug_RAM.ini: Initialization File for Debugging from Internal RAM */
/******************************************************/
/* This file is part of the uVision/ARM development tools. */
/* Copyright (c) 2005-2014 Keil Software. All rights reserved. */
/* This software may only be used under the terms of a valid, current, */
/* end user licence from KEIL for a compatible version of KEIL software */
/*development tools. Nothing else gives you the right to use this software */
/***************************************************/

FUNC void Setup (void) {
SP = _RDWORD(0x20000000);               // 设置栈指针SP，把0x20000000地址中的内容赋值到SP。
PC = _RDWORD(0x20000004);               // 设置程序指针PC，把0x20000004地址中的内容赋值到PC。
_WDWORD(0xE000ED08, 0x20000000);        // Setup Vector Table Offset Register
}

LOAD %L INCREMENTAL                     // 下载 axf 文件到RAM
Setup();                                //调用上面定义的 setup 函数设置运行环境
```

### 程序调试下载配置

打开工程配置窗口，切换到 Debug 一栏，选择对应的调试器，然后点击 Setting 按钮，在打开的设置界面勾选 Verify Code DownLoad 和 DownLoad to Flash 两项。

![](/image/2025/stm32-sram-debug/new-debug07.jpg) ![](/image/2025/stm32-sram-debug/new-debug08.jpg)

在上一步打开的工程配置窗口 Debug-Setting 选择 Flash Download 一栏，在左上方选择 Do not Erase 选项，同时在右侧配置写入 SRAM 的地址信息，这里的信息必须和 Target 中配置的信息一致；

![](/image/2025/stm32-sram-debug/new-debug09.jpg) ![](/image/2025/stm32-sram-debug/new-debug10.jpg)

配置结束后，直接进行编译、调试程序即可。
