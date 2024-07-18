---
title: Linux Rpm 打包流程
date: 2020-08-10 09:05:00
tags: [Linux, Programming]
categories:
  - 软件开发总结
---

RedHat 系列及分支的安装包（.rpm）的打包流程

<!-- more -->
## 准备工作

1. 因为打 RPM 包需要 rpmbuild 命令，所以需要先安装 rpmbuild 相应工具，安装过程可在网上查找；
2. 输入命令 rpmbuild xx.spec 即可自动在当前用户的 $HOME 目录下创建 RPM 相关的文件夹，也可手动创建。

输入命令：

``` BASH
rpmbuild xx.spec
```

命令运行后，输入：

``` BASH
tree ~/rpmbuild/
```

即可显示：

``` BASH
rpmbuild/
├── BUILD
├── BUILDROOT
├── RPMS
├── SOURCES
├── SPECS
└── SRPMS
```

- BUILD：源码包被解压至此，并在该目录的子目录完成编译，宏为 %_builddir
- BUILDROOT：保存 %install 阶段安装的文件，宏为 %_buildrootdir
- RPMS：生成/保存二进制 RPM 包，宏为 %_rpmdir
- SOURCES：保存源码包（如 .tar 包）和所有 patch 补丁，宏为 %_sourcedir
- SPECS：保存 RPM 包配置（.spec）文件，宏为 %_specdir
- SRPMS：生成/保存源码 RPM 包(SRPM)，宏为 %_srcrpmdir
- ~/rpmbuild 的宏为 %_topdir

## 编辑 SPEC 文件

1. 运行以下命令创建 .spec 文件：

``` BASH
rpmdev-newspec xxx.spec
```

2. 将 SPEC 文件放在 ~rpmbuild/SPECS/ 下面，然后编辑 SPEC 文件，如下所示：

``` BASH
Name:　　　　#软件名称
Version:　　#版本号
Release:　　#发布编码
Summary:　　#简要说明
License:　　#协议版本
URL:
Source0:　　#源码包
%description
#描述

%prep
#预处理

%build
#编译

%pre
#安装前

%install
#安装

%post
#安装后

%files
#安装的文件列表

%clean
#清理临时文件

%preun
#卸载前

%postun
#卸载后

%changelog
#修改历史
* Fri Aug  7 2020 
-
```

## 安装的各个阶段说明

- %prep阶段 - 预处理，主要对源代码包进行解压和打补丁
    一般使用  %setup  -c 或者 %setup -q 命令来解压源码包，直接会将文件解压到%{_builddir}
- %build阶段 - 对源代码包进行编译
    编译阶段，非 GNU configure 配置的程序可以不关注这个阶段
- %install阶段 - 将软件安装到虚拟根目录, 同时 Install 阶段也有如下阶段
    - %pre阶段 - 安装前  
        > $1 == 1 代表安装  
        > $1 == 2 代表升级  
    - %post阶段 - 安装后  
        > $1 == 1 代表安装  
        > $1 == 2 代表升级  
    - %preun阶段 - 卸载前  
        > $1 == 0 代表卸载  
        > $1 == 1 代表升级  
    - %postun阶段 - 卸载后  
        > $1 == 0 代表卸载  
        > $1 == 1 代表升级  
        这个阶段主要从 %{_builddir} 复制相关文件到 %{buildroot} （虚拟根目录）目录，如下所示：  
        > rm -rf $RPM_BUILD_ROOT  
        > cp -rf xxx $RPM_BUILD_ROOT
- files 阶段-列出被打包的文件和目录
    首先要设定默认权限，同时要列出打包的目录和文件，设定默认权限的命令如下：
    ``` BASH
    %defattr(<文件权限>, <用户>, <用户组>, <目录权限>)
    ```
    第 4 个参数通常会省略。常规用法为 %defattr(-,root,root,-)，其中 “-” 表示默认权限。
    在列出文件的目录时，尽量使用内建宏来代替目录名。

### 常用的内建宏
``` BASH
%{_sysconfdir}        /etc
%{_prefix}            /usr
%{_exec_prefix}       %{_prefix}
%{_bindir}            %{_exec_prefix}/bin
%{_lib}               lib (lib64 on 64bit systems)
%{_libdir}            %{_exec_prefix}/%{_lib}
%{_libexecdir}        %{_exec_prefix}/libexec
%{_sbindir}           %{_exec_prefix}/sbin
%{_sharedstatedir}    /var/lib
%{_datadir}           %{_prefix}/share
%{_includedir}        %{_prefix}/include
%{_oldincludedir}     /usr/include
%{_infodir}           /usr/share/info
%{_mandir}            /usr/share/man
%{_localstatedir}     /var
%{_initddir}          %{_sysconfdir}/rc.d/init.d 
%{_topdir}            %{getenv:HOME}/rpmbuild
%{_builddir}          %{_topdir}/BUILD
%{_rpmdir}            %{_topdir}/RPMS
%{_sourcedir}         %{_topdir}/SOURCES
%{_specdir}           %{_topdir}/SPECS
%{_srcrpmdir}         %{_topdir}/SRPMS
%{_buildrootdir}      %{_topdir}/BUILDROOT
%{_var}               /var
%{_tmppath}           %{_var}/tmp
%{_usr}               /usr
%{_usrsrc}            %{_usr}/src
%{_docdir}            %{_datadir}/doc
%{buildroot}          %{_buildrootdir}/%{name}-%{version}-%{release}.%{_arch}
$RPM_BUILD_ROOT       %{buildroot}
```
- %clean阶段 - 完成后的一些清理工作
    主要是清理 %{_builddir}和%{_buildrootdir}两个目录里的中间文件
- %changelog阶段 -- 主要记录每次打包时的修改日志
``` BASH
%changelog
* Fri Aug  7 2020 - Your Name <youremail@xxx.xxx> - Release
- Update log1 
* Fri Aug  7 2020 - Your Name <youremail@xxx.xxx> - Release
- Update log2
```

## 运行 RPMBUILD 命令完成打包

在 SPEC 目录下执行 rpmbuild -xx xxx.spec 命令完成打包，rpmbuild 命令选项如下所示：
``` BASH
#rpmbuild
-bp 预处理
-bc 编译
-bi 编译并安装
-bl 检验文件是否齐全
-ba 编译后做成*.rpm和src.rpm
-bb 编译后做成*.rpm
-bs 只做成*.src.rpm
```
