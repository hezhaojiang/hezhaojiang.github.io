---
title: 树莓派日记 - 安装 rtmpdump
categories:
  - 树莓派日记
tag:
    - rtmpdump
abbrlink: 5ff634f
date: 2020-10-19 14:17:30
---
`rtmpdump` 是一个 `C++` 的开源工程，也可以说是一个用来处理 `RTMP` 流媒体的工具包，支持 `rtmp://`, `rtmpt://`, `rtmpe://`, `rtmpte://`, 和 `rtmps://` 等。我们可以通过对 `rtmpdump` 的学习来学习 `RTMP` 协议。

首先可以了解一下其使用方法：`rtmpdump` 使用说明：

官网：<http://rtmpdump.mplayerhq.hu/>

<!--more-->

## 下载源码

``` bash
$ git clone git://git.ffmpeg.org/rtmpdump && cd rtmpdump
```

## 打补丁包

对于 `openssl 1.1` 以上的 `linux` 用户，直接编译无法编译通过，需要打补丁包对 `openssl` 的接口进行一些修改，并消除了部分编译警告。

补丁包下载链接如下：

[点击下载](/slave/Openssl-1.1.1.patch "补丁下载")

补丁包使用方法：

将 `patch` 文件放在源码同一目录下，执行以下命令：

```bash
$ git am Openssl-1.1.1.patch
Applying: 【MOD】支持 Openssl 1.1.1
```

## 编译 & 安装

``` bash
$ make && make install
```

## 参考资料

* [1] [rtmpdump](http://rtmpdump.mplayerhq.hu/)
* [2] [JudgeZarbi/RTMPDump-OpenSSL-1.1: RTMPDump modified to compile with the changes made in OpenSSL 1.1.0](https://github.com/JudgeZarbi/RTMPDump-OpenSSL-1.1)
