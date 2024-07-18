---
title: Linux Deb 打包流程
date: 2020-08-10 17:46:00
tags: [Linux, Programming]
categories:
  - 软件开发总结
---

Debian 系列及分支的安装包（.deb）的打包流程

<!-- more -->
## 准备工作

1. 准备打包的二进制文件
2. 建立一个虚拟根目录，里面包含 DEBIAN 目录和软件安装路径，如下所示：

``` BASH
soft-name
    |--DEBIAN
    |       |--control
    |       |--postinst
    |       |--postrm
    |       |--preinst
    |       |--prerm
    |       |--copyright
    |
    |--opt
        |--softposition
```

control 主要用来描述软件的版本，名称等详细信息，如下所示：

``` BASH
Package:
Version:
Description:
Section:
Priority:
Architecture:
Installed-Size:
Depends:
Pre-Depends:
Maintainer:
```

- Package -- 软件包名称
- Version -- 版本号
- Description -- 软件描述
- Section -- 软件类型 utils, net, mail, text, x11
- Priority -- 软件对系统的重要程度，required, standard, optional, extra 等
- Architecture -- 软件支持的平台，如 amd64 arm64 等
- Installed-Size -- 软件尺寸
- Depends -- 软件依赖的其他软件和库文件等，多个文件用逗号隔开
- Pre-Depends -- 安装软件前需要安装的库或软件
- Maintainer -- 打包者信息或者联系方式

安装过程中各个脚本的调用次序如下, 这些脚本均为 bash shell：
- preinst 文件于软件包安装之前会被调用
- postinst 文件于软件包安装之后被调用
- prerm 文件于软件包卸载之前调用
- postrm 文件于软件包卸载之后调用

## 运行打包命令

编写完脚本之后，运行：

``` BASH
dpkg-deb -b soft-name soft-name.deb
```
