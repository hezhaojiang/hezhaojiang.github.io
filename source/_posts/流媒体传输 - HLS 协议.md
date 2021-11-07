---
title: 流媒体传输 - HLS 协议
categories:
  - 流媒体传输
tags:
  - HLS
abbrlink: f5bbcc60
date: 2020-11-22 14:41:05
---
`HLS` 全称是 `HTTP Live Streaming`，是一个由 `Apple` 公司提出的基于 `HTTP` 的媒体流传输协议，用于实时音视频流的传输。目前 `HLS` 协议被广泛的应用于视频点播和直播领域。

<!-- more -->

## 概述

### 原理介绍

通过将整条流切割成一个小的可以通过 `HTTP` 下载的媒体文件, 然后提供一个配套的媒体列表文件, 提供给客户端, 让客户端顺序地拉取这些媒体文件播放, 来实现看上去是在播放一条流的效果. 由于传输层协议只需要标准的 `HTTP` 协议, HLS 可以方便的透过防火墙或者代理服务器, 而且可以很方便的利用 CDN 进行分发加速, 并且客户端实现起来也很方便.

### 整体架构

`HLS` 的架构分为三部分：Server，CDN，Client 。即服务器、分发组件和客户端。

下面是 `HLS` 整体架构图：

![HLS 整体架构图](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201122225230.png)

服务器用于接收媒体输入流，对它们进行编码，封装成适合于分发的格式，然后准备进行分发。

分发组件为标准的 Web 服务器。它们用于接收客户端请求，传递处理过的媒体，把资源和客户端联系起来。

客户端软件决定请求何种合适的媒体，下载这些资源，然后把它们重新组装成用户可以观看的连续流。

## HLS 协议分析

### HLS Playlist

其实, `HLS` 协议的主要内容是关于 `M3U8` 这个文本协议的, 其实生成与解析都非常简单. 为了更加直接地说明这一点, 我下面举两个简单的例子:

简单的 `Media Playlist`：

    #EXTM3U
    #EXT-X-VERSION:3
    #EXT-X-TARGETDURATION:8
    #EXT-X-MEDIA-SEQUENCE:2680
    #EXTINF:7.975,
    https://priv.example.com/fileSequence2680.ts
    #EXTINF:7.941,
    https://priv.example.com/fileSequence2681.ts
    #EXTINF:7.975,
    https://priv.example.com/fileSequence2682.ts

包含多种比特率的 `Master Playlist`：

    #EXTM3U
    #EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=1280000
    http://example.com/low.m3u8
    #EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=2560000
    http://example.com/mid.m3u8
    #EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=7680000
    http://example.com/hi.m3u8
    #EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=65000,CODECS="mp4a.40.5"
    http://example.com/audio-only.m3u8

- `HLS` 通过 `URI(RFC3986)` 指向的一个 `Playlist` 来表示一个媒体流
- 一个 `Playlist` 可以是一个 `Media Playlist` 或者 `Master Playlist`, 使用 `UTF-8` 编码的文本文件, 包含一些 `URI` 跟描述性的 `tags`
- 一个 `Media Playlist` 包含一个 `Media Segments` 列表,当顺序播放时, 能播放整个完整的流
- 要想播放这个 `Playlist`, 客户端需要首先下载他, 然后播放里面的每一个 `Media Segment`
- 更加复杂的情况是, `Playlist` 是一个 `Master Playlist`, 包含一个 `Variant Stream` 集合, 通常每个 `Variant Stream` 里面是同一个流的多个不同版本(如: 分辨率, 码率不同)

- 一个 `Playlist` 文件必须通过 `URI(.m3u8 或 m3u)` 或者 `HTTP Content-Type` 来识别 (`application/vnd.apple.mpegurl` 或 `audio/mpegurl`)
- 换行符可以用 `\n` 或者 `\r\n`
- 以 `#` 开头的是 `tag` 或者注释, 以 `#EXT` 开头的是 `tag`, 其余的为注释, 在解析时应该忽略
- `Playlist` 里面的 `URI` 可以用绝对地址或者相对地址, 如果使用相对地址, 那么是相对于 `Playlist` 文件的地址

### HLS Media Segments

- 每一个 `Media Segment` 通过一个 `URI` 指定, 可能包含一个 `byte range`
- 每一个 `Media Segment` 的 `duration` 通过 `EXTINF` `tag` 指定
- 每一个 `Media Segment` 有一个唯一的整数 `Media Segment Number`
- 有些媒体格式需要一个 `format-specific sequence` 来初始化一个 `parser`, 在 `Media Segment` 被 `parse` 之前. 这个字段叫做 `Media Initialization Section`, 通过 `EXT-X-MAP` `tag` 来指定. 支持的 `Media Segment` 格式

### HLS TAGS

- `Basic Tags` : 用在 `Media Playlist` 和 `Master Playlist` 里面
    - `EXTM3U`: 必须在文件的第一行, 标识是一个 `Extended M3U Playlist` 文件
    - `EXT-X-VERSION`: 表示 `Playlist` 兼容的版本
- `Media Segment Tags` : 只能出现在 `Media Playlist` 里面
    - `EXTINF`: 用于指定 `Media Segment` 的 `duration`
    - `EXT-X-BYTERANGE`: 用于指定 `URI` 的 `sub-range`
    - `EXT-X-DISCONTINUITY`: 表示不连续
    - `EXT-X-KEY`: 表示 `Media Segment` 已加密, 该值用于解密
    - `EXT-X-MAP`: 用于指定 `Media Initialization Section`
    - `EXT-X-PROGRAM-DATE-TIME`: 和 `Media Segment` 的第一个 `sample` 一起来确定时间戳
    - `EXT-X-DATERANGE`: 将一个时间范围和一组属性键值对结合到一起
- `Media Playlist tags` : 只能出现在 `Media Playlist` 里面
    - `EXT-X-TARGETDURATION`: 用于指定最大的 `Media Segment duration`
    - `EXT-X-MEDIA-SEQUENCE`: 用于指定第一个 `Media Segment` 的 `Media Sequence Number`
    - `EXT-X-DISCONTINUITY-SEQUENCE`: 用于不同 `Variant Stream` 之间同步
    - `EXT-X-ENDLIST`: 表示结束
    - `EXT-X-PLAYLIST-TYPE`: 可选, 指定整个 `Playlist` 的类型
    - `EXT-X-I-FRAMES-ONLY`: 表示每个 `Media Segment` 描述一个单一的 `I-frame`
- `Master Playlist tags` : 只能出现在 `Master Playlist` 中
    - `EXT-X-MEDIA`: 用于关联同一个内容的多个 `Media Playlist` 的多种 `renditions`
    - `EXT-X-STREAM-INF`: 用于指定一个 `Variant Stream`
    - `EXT-X-I-FRAME-STREAM-INF`: 用于指定一个 `Media Playlist` 包含媒体的 `I-frames`
    - `EXT-X-SESSION-DATA`: 存放一些 `session` 数据
    - `EXT-X-SESSION-KEY`: 用于解密
- `Media or Master Playlist Tags` : 可以出现在 `Media Playlist` 或者 `Master Playlist` 中，但是如果同时出现在同一个 `Master Playlist` 和 `Media Playlist` 中时，必须为相同值
    - `EXT-X-INDEPENDENT-SEGMENTS`: 表示每个 `Media Segment` 可以独立解码
    - `EXT-X-START`: 标识一个优选的点来播放这个 `Playlist`

## HLS 播放

### 播放未加密 HLS

我们通过 `VLC` 播放器播放苹果官方提供的一个例子：`http://devimages.apple.com/iphone/samples/bipbop/gear1/prog_index.m3u8`，并使用 `Wirshark` 对其中交互进行抓包。

    GET /iphone/samples/bipbop/gear1/prog_index.m3u8 HTTP/1.1
    Host: devimages.apple.com
    Accept: */*
    Accept-Language: zh_CN
    User-Agent: VLC/3.0.8 LibVLC/3.0.8
    Range: bytes=0-

        HTTP/1.1 206 Partial Content
        Accept-Ranges: bytes
        Content-Type: audio/x-mpegurl
        ETag: "50117c8233644c19b5ab49551b72507f:1239907352"
        Last-Modified: Thu, 16 Apr 2009 18:42:32 GMT
        Server: AkamaiNetStorage
        Date: Sun, 22 Nov 2020 15:01:49 GMT
        Content-Range: bytes 0-7018/7019
        Content-Length: 7019
        X-Cache: TCP_MEM_HIT from a184-26-91-45.deploy.akamaitechnologies.com (AkamaiGHost/10.2.0.2-31441410) (-)
        Connection: keep-alive

        #EXTM3U
        #EXT-X-TARGETDURATION:10
        #EXT-X-MEDIA-SEQUENCE:0
        #EXTINF:10, no desc
        fileSequence0.ts
        #EXTINF:10, no desc
        fileSequence1.ts
        #EXTINF:10, no desc
        fileSequence2.ts
        #EXTINF:10, no desc
        ......
        #EXTINF:1, no desc
        fileSequence180.ts
        #EXT-X-ENDLIST

我们可以看到返回的 `M3U8` 里面他有一个一个 `ts` 视频片段，这个一个一个视频片段就是我们需要的播放的视频片段。

`#EXTINF` 表示每个 `ts` 切片视频文件的时长。
`#EXT-X-TARGETDURATION` 指定当前视频流中的切片文件的最大时长，也就是说这些 `ts` 切片的时长不能大于 `#EXT-X-TARGETDURATION` 的值。
`#EXT-X-MEDIA-SEQUENCE` 第一个 `ts` 分片的序列号
`#EXT-X-ENDLIST` 这个表示视频结束，有这个标志同时也说明当前的流是一个非直播流。
`#EXT-X-PLAYLIST-TYPE:VOD` 的意思是当前的视频流并不是一个直播流，而是点播流，换句话说就是该视频的全部的 `ts` 文件已经被生成好了
`#EXT-X-ALLOW-CACHE` 是否允许 `cache`

### 播放加密 HLS

## HLS 协议总结

### 优点

- 客户端支持简单, 只需要支持 `HTTP` 请求即可, `HTTP` 协议无状态, 只需要按顺序下载媒体片段即可。
- 使用 `HTTP` 协议网络兼容性好, `HTTP` 数据包也可以方便地通过防火墙或者代理服务器, `CDN` 支持良好。
- `Apple` 的全系列产品支持，不需要安装任何插件就可以原生支持播放 `HLS`, 目前 `Android` 也加入了对 `HLS` 的支持。
- 自带多码率自适应机制。

### 缺点

- 相比 `RTMP` 这类长连接协议, 延时较高, 难以用到互动直播场景。
- 对于点播服务来说, 由于 `TS` 切片通常较小, 海量碎片在文件分发, 一致性缓存, 存储等方面都有较大挑战。

### 改进

- 由于客户端每次请求 `TS` 或 `M3U8` 有可能一个新的连接请求, 无法有效的标识客户端, 一旦出现问题, 基本无法有效的定位问题。
- 一般工业级的服务器都会对传统的 `HLS` 做一些改进，常见优化是对每个 `M3U8` 文件增加 `Session` 来标识一条 `HLS` 连接。
- 不管通过哪种方式, 最终我们都能通过一个唯一的 `id` 来标识一条流, 这样在排查问题时就可以根据这个 `id` 来定位播放过程中的问题。

## 参考资料

* [1] [流媒体协议（一）：HLS 协议 - 灰色飘零 - 博客园](https://www.cnblogs.com/renhui/p/8081869.html)
