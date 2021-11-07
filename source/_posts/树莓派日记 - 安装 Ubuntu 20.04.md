---
title: 树莓派日记 - 安装 Ubuntu 20.04
categories:
    - 树莓派日记
tag:
    - ubuntu
abbrlink: 2e43eec
date: 2020-07-23 14:33:36
---
> 树莓派开不开机了，身边暂时没有显示器，所以打算直接重装系统吧……
> 记录以下装系统过程，方便以后再发生如此问题（希望不会）

树莓派安装 Ubuntu 的作用：

1. 作为流媒体服务器学习音视频传输技术
2. 搭建 Web 服务器学习 Nginx 原理
3. 学习 Linux 基本操作和 Shell 脚本
4. 尝试简单的内核级开发

<!-- more -->

## 准备工作

### 硬件准备工作

1. 树莓派 4b
2. Micro SD 卡（32G ~ 128G 容量，Class 10 以上）
3. SD 卡读卡器
4. 个人计算机

### 软件准备工作

* Ubuntu 20.04 镜像
    > 下载地址：<https://ubuntu.com/download/raspberry-pi>
* balenaEtcher（用于刻录镜像）
    > 下载地址：<https://github.com/balena-io/etcher/releases>

## 安装过程

### What you’ll need

- A Raspberry Pi 2, 3, or 4
- A micro-USB power cable for the Pi 2 & 3, or a USB-C power cable for the Pi 4
- A microSD card with the Ubuntu Server image
- A monitor with an HDMI interface
- An HDMI cable for the Pi 2 & 3 and a MicroHDMI cable for the Pi 4
- A USB keyboard

### Flash Ubuntu onto your microSD card

The first thing you need to do is take a minute to copy the Ubuntu image on to a microSD card by following our tutorials, we have one for Ubuntu machines, Windows machines and Macs.

### Boot Ubuntu Server

1. You need to attach a monitor and keyboard to the board. You can alternatively use a serial cable.
2. Now insert the microSD card
3. Plug the power adaptor into the board

### Login to your Pi

When prompted to log in, use `ubuntu` for the username and the password. You will be asked to change this default password after you log in.

You are now running the Ubuntu Server on your Raspberry Pi.

## 基础环境配置过程

### 换源

Ubuntu 安装后默认使用官方源，服务器位于国外，网络环境较差，推荐更换国内阿里源：

``` bash
$ sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak_`date "+%y_%m_%d"`
$ sudo sed -i 's/http:\/\/ports.ubuntu.com/https:\/\/mirrors.aliyun.com/g' /etc/apt/sources.list
$ sudo apt-get update
$ sudo apt-get upgrade
```

### 安装常用依赖

``` bash
$ sudo apt-get install libssl-dev
```

### 安装常用工具

``` bash
$ # sudo apt-get install net-tools # 已启用，建议使用 iproute2
$ sudo apt-get install samba # 需要配置
$ sudo apt-get install lrzsz
```

### 安装开发环境

``` bash
$ sudo apt-get install build-essential
$ sudo apt-get install cmake cmake-curses-gui
$ sudo apt-get install python3 python3-dev
```

## 进阶环境配置过程

### Web

### 安装 Nginx

详见：{% post_link "Nginx - 环境搭建" %}

### 音视频

#### 安装 ffmpeg

详见：{% post_link "树莓派日记 - 安装 ffmpeg" %}

#### 安装 SRS

详见：{% post_link "树莓派日记 - 安装 SRS" %}

## 参考资料

* [1] [Install Ubuntu Server on a Raspberry Pi 2, 3 or 4 | Ubuntu](https://ubuntu.com/download/raspberry-pi)
