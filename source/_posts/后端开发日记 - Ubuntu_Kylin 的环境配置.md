---
title: 后端开发日记 - Ubuntu_Kylin 的环境配置
categories:
  - 后端开发日记
tags:
  - ubuntu
abbrlink: a9e409e6
date: 2020-04-24 12:02:16
---

<!-- toc -->

## 1. 更换国内源

首先要查看ubuntu版本代号

```bash
$ lsb_release -c
Codename: bionic
```

<!-- more -->

然后修改文件

```bash
/* 新系统没有vim 先凑合一下 */
$ sudo vi /etc/apt/sources.list
```

修改内容为：

```bash
# 要注意 bionic 字段要和 ubuntu 版本代号匹配
# 阿里源
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

执行

```bash
# 更新软件列表
$ sudo apt-get update
# 更新软件
$ sudo apt-get upgrade
```

## 2. 安装常用软件

```bash
# vim
$ sudo apt-get install vim
$ sudo apt-get install cscope
$ sudo apt-get install ctags
# git
$ sudo apt-get install git
```

## 3. 安装常用开发环境

```bash
# Kernel
$ sudo apt-get install libncurses5-dev
$ sudo apt-get install libncursesw5-dev
$ sudo apt-get install ncurses-dev
$ sudo apt-get install bison
$ sudo apt-get install flex
# C/C++
$ sudo apt-get install build-essential
$ sudo apt-get install cmake  
# ssl
$ sudo apt-get install libssl-dev
$ sudo apt-get install openssl
# arm
$ sudo apt-get install qemu
# arm-gcc
$ sudo apt-get install gcc-arm-linux-gnueabi
$ sudo apt-get install gcc-5-arm-linux-gnueabi
# update-alternatives 选择 gcc 版本
$ sudo update-alternatives --install /usr/bin/arm-linux-gnueabi-gcc  arm-linux-gnueabi-gcc  /usr/bin/arm-linux-gnueabi-gcc-5 5
$ sudo update-alternatives --install /usr/bin/arm-linux-gnueabi-gcc  arm-linux-gnueabi-gcc  /usr/bin/arm-linux-gnueabi-gcc-7 7
$ sudo update-alternatives --config arm-linux-gnueabi-gcc
# python
$ sudo apt-get install python-dev
$ sudo apt-get install python3-dev
# gdb-multiarch
$ sudo apt-get install gdb-multiarch
```
