---
title: 音视频日记 - ffmpeg 命令行
categories:
  - 音视频日记
tags:
  - ffmpeg
abbrlink: 4765df61
date: 2021-01-04 14:38:32
---
`ffmpeg` 是一个基于 `ffmpeg` 库的经典转码程序，几乎被所有的知名系统、软件工具、服务器后台等静默使用，通过命令行启动转码，这里主要以命令行语法讲解为主。

<!-- more -->

## 查看 ffmpeg 帮助命令

在学习任何一个命令行程序前，首先要知道如何查看对应命令的帮助，在一般情况下，`linux` 平台下程序的帮助信息是通过 `程序名 -h` 的方式进行输出，`ffmpeg` 亦是如此。

精简帮助命令：`ffmpeg –h`
更多帮助命令：`ffmpeg –h long`
完整帮助命令：`ffmpeg –h full`

## ffmpeg 的语法格式

ffmpeg 的语法格式如下所示

    ffmpeg [输入源参数] -i [输入 URL] [输出参数] [输出 URL]

    其中 [输入源参数] 和 [输出参数] 的语法格式为：
    [options] [value(可以省略)] ... ...

示例 1：

    ffmpeg -f mpegts -i "http://AVTestFile/AVNormal/52" -vcodec x264Encoder -r 15 -b:v 256000 -vf scale=800:600 -an -copyts -y "51.avi"
    * 输入源使用 mpegts 容器 http 协议的 URL
    * -vcodec x264Encoder 使用 x264Encoder 视频编码器
    * -r 15 视频帧率 15 fps
    * -b:v 256000 视频编码码率 265Kbps
    * -vf scale=800:600 使用视频滤器 scale 进行缩放到 800x600 尺寸
    * -an 禁用音频
    * -copyts 时间戳拷贝
    * -y 覆盖输出文件
    * 输出 URL 默认使用 file 协议，输出到文件 51.avi

示例 2：

    ffmpeg -i BaiCaoYuan.mp4 -ss 00:00:19 -t 00:06:48 -dcodec copy -b:v 4000K -y cut.mp4
    * 输入源使用 file 协议，为 BaiCaoYuan.mp4 文件
    * -dcodec copy 使用与源视频一致的编解码器
    * -b:v 4000K 视频编码码率 4000Kbps
    * -y 覆盖输出文件
    * 输出 URL 默认使用 file 协议，输出到文件 cut.mp4

## 常用参数

| 参数 | 示例 | 说明 |
| - | - | - |
| -t duration | -t 00:06:48 | 设置处理时间，格式为hh:mm:ss |
| -ss position | -ss 00:00:19 | 设置起始时间，格式为hh:mm:ss |
| -b:v bitrate | -b:v 256000 | 设置视频码率 |
| -b:a bitrate | -b:a 320000 | 设置音频码率 |
| -r fps | -r 25 | 设置帧率 |
| -s wxh | -s 800x600 | 设置帧大小，格式为WxH |
| -c:v codec | -c:v h264 |设置视频编码器 |
| -c:a codec | -c:a aac |设置音频编码器 |
| -ar freq | -ar 44100 | 设置音频采样率 |

## 参考资料

* [1] 基于 FFmpeg + SDL 的视频播放器的制作 -- 雷霄骅
* [2] 多媒体编程开发之 FFmpeg 基础库
