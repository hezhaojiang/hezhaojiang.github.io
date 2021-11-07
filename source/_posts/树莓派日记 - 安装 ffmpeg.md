---
title: 树莓派日记 - 安装 ffmpeg
categories:
    - 树莓派日记
tag:
    - ffmpeg
abbrlink: c30ccf98
date: 2020-07-25 14:49:01
---

## 环境准备

* Ubuntu Server 20.04 LTS

> 安装过程详见：{% post_link "树莓派日记 - 安装 Ubuntu 20.04" %}

<!-- more -->

## 依赖环境

### cmake

``` bash
# For Ubuntu
$ sudo apt-get install -y cmake cmake-curses-gui
```

### gcc/g++

``` bash
# For Ubuntu
$ sudo apt-get install -y build-essential
```

### 其他

``` bash
# For Ubuntu
$ sudo apt-get install -y pkg-config
```

## 安装方式一：包管理器安装

`Ubuntu` 系统通过 `apt` 包管理器安装 `ffmpeg` 较为方便，依次执行以下命令即可完成安装

``` bash
$ sudo apt-get install -y libsdl-dev libavcodec-dev libavutil-dev
$ sudo apt-get install ffmpeg ffmpeg-devel
```

对于 `CentOS` 系统，`yum` 包管理器中并没有 `ffmpeg` 包，需要手动添加额外的

``` bash
yum install -y epel-release
sudo rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro

# CentOS 7 执行以下命令
sudo rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm

# CentOS 7 执行以下命令
sudo rpm -Uvh http://li.nux.ro/download/nux/dextop/el6/x86_64/nux-dextop-release-0-2.el6.nux.noarch.rpm

yum repolist
yum install -y ffmpeg ffmpeg-devel
```

## 安装方式二：源码安装

### 安装依赖库

``` bash
# nasm, x264 依赖 nasm 的汇编加速
$ wget https://www.nasm.us/pub/nasm/releasebuilds/2.15rc12/nasm-2.15rc12.tar.xz
$ tar -xJf nasm-2.15rc12.tar.xz
$ cd nasm-2.15rc12/
$ ./configure && make && sudo make install

# x264, 建议安装 x264 的 H264 编码质量较高
$ git clone https://code.videolan.org/videolan/x264.git
$ cd x264
$ ./configure --enable-shared --enable-static && make && sudo make install

# yasm, ffmpeg 依赖 yasm
$ wget http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz
$ tar xzf yasm-1.3.0.tar.gz
$ cd yasm-1.3.0/
$ sed -i 's#) ytasm.*#)#' Makefile.in
$ ./configure --prefix=/usr && make && sudo make install

# 以下依赖库可选安装

# x265
# get x265 at http://ftp.videolan.org/pub/videolan/x265/
$ tar xzf x265_2.3.tar.gz
$ cd x265_2.3/build/linux/
$ ./make-Makefiles.bash     # 注意将 ENABLE_PIC 设置为 ON
$ make && sudo make install

# fdk-aac
$ git clone https://github.com/mstorsjo/fdk-aac.git
$ cd fdk-aac
$ sudo apt install -y autoconf libtool
$ ./configure CFLAGS=-fPIC && make && sudo make install
```

### 编译 & 安装

``` bash
$ git clone https://github.com/FFmpeg/FFmpeg.git ffmpeg
$ cd ffmpeg/
$ ./configure --enable-libx264 --enable-gpl
$ make -j4 && sudo make install
```

### 配置

``` bash
$ sudo echo "/usr/local/x264/lib/" > /etc/ld.so.conf.d/ffmpeg.conf
$ ldconfig
```

## 参考资料
