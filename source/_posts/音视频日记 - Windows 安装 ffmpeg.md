---
title: 音视频日记 - Windows 安装 ffmpeg
categories:
  - 音视频日记
tags:
  - ffmpeg
abbrlink: a1b64acb
date: 2021-01-03 12:00:27
---
在 `Windows` 上编译 `ffmpeg` 没有在 `linux` 上简单方便，在此记录编译过程

<!-- more -->

## 配置 MSVC 编译环境

配置环境变量如下（VS2019）：

``` bash
# PATH
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29333\bin\Hostx64\x64\
# INCLUDE
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29333\include
C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\shared
C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\ucrt
C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\um
C:\Program Files (x86)\Windows Kits\10\Include\10.0.18362.0\winrt
# LIB
C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Tools\MSVC\14.28.29333\lib\x64
C:\Program Files (x86)\Windows Kits\10\Lib\10.0.18362.0\um\x64
C:\Program Files (x86)\Windows Kits\10\Lib\10.0.18362.0\ucrt\x64
```

## MSYS2 的安装与配置

``` bash
# 找到以下内容
rem set MSYS2_PATH_TYPE=inherit
# 修改为
set MSYS2_PATH_TYPE=inherit
```

执行 `cl` 命令查看输出：

``` bash
# cl
用于 x64 的 Microsoft (R) C/C++ 优化编译器 19.28.29335 版
版权所有(C) Microsoft Corporation。保留所有权利。

用法: cl [ 选项... ] 文件名... [ /link 链接选项... ]
```

如果是以上返回则配置成功

如果提示 `bash: cl: 未找到命令` 则需要检查是不是上述配置出了问题，或者重启电脑再试一下。

## 安装依赖环境

``` bash
# 如果网络连接不上，可以尝试使用代理
export http_proxy="127.0.0.1:10809"
export https_proxy="127.0.0.1:10809"
# 安装以下依赖
pacman -S nasm
pacman -S yasm
pacman -S make cmake
pacman -S diffutils
pacman -S pkg-config
pacman -S git
pacman -S mingw-w64-x86_64-SDL2
```

## 编译 x264

## 编译 x265

## 编译 ffmpeg

``` bash
./configure --enable-yasm --enable-asm --toolchain=msvc --enable-shared --enable-static
make -j
make install
```
