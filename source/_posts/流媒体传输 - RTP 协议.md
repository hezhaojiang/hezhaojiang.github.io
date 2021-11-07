---
title: 流媒体传输 - RTP 协议
categories:
    - 流媒体传输
tags:
  - RTP
abbrlink: faccfa50
date: 2020-07-28 14:15:42
---
## RTP 协议

### RTP Header

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |V=2|P|X|  CC   |M|     PT      |       sequence number         |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                           timestamp                           |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           synchronization source (SSRC) identifier            |
    +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
    |            contributing source (CSRC) identifiers             |
    |                             ....                              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

<!-- more -->

前 12 个字节出现在每个 RTP 数据包中，这些字段具有的含义如下：

* **版本 (V)**：2 位，该字段标识 RTP 的版本。RFC3550 定义的版本规范为 2.

* **填充 (P)**：1 位，如果设置了填充位，则数据包包含一个或多个末尾的其他填充字节不属于有效载荷。

* **扩展名 (X)**：1 位，如果扩展位被置位，则固定报头必须后跟一个扩展头部。

* **CSRC 计数 (CC)**：4 位，CSRC 计数包含随后的 CSRC 标识符的数量固定头。

* **标记 (M)**：1 位，由配置文件定义，它允许诸如框架边界之类的重要事件发生在数据包流中被标记。

> **标记 (M)**：1 位，标识此 RTP 包携带了一组 NAL 数据的最后一个 NAL Unit。（RFC 6184）

* **有效载荷类型 (PT)**：7 位，该字段标识 RTP 有效负载的格式并确定由应用程序对其进行解释。一个配置文件可以指定一个负载类型代码到负载格式的默认静态映射。

* **序列号 (sequence number)**：16 位，每个 RTP 数据包的序列号加 1 发送，接收方可以使用它来检测丢包并恢复报文序列，序号的初始值应该是随机的。

* **时间戳 (timestamp)**：32 位，时间戳记录了该包中数据的第一个字节的采样时刻。在一次会话开始时，时间戳初始化成一个初始值。即使在没有信号发送时，时间戳的数值也要随时间而不断地增加（时间在流逝嘛）。时间戳是去除抖动和实现同步不可缺少的。

* **同步源标识符 (SSRC)**：32 位，同步源就是指 RTP 包流的来源。在同一个 RTP 会话中不能有两个相同的 SSRC 值。该标识符是随机选取的 RFC1889 推荐了 MD5 随机算法。

* **贡献源列表 (CSRC List)**：0～15 项，每项 32 位，用来标志对一个 RTP 混合器产生的新包有贡献的所有 RTP 包的源。由混合器将这些有贡献的 SSRC 标识符插入表中。SSRC 标识符都被列出来，以便接收端能正确指出交谈双方的身份。

> [RFC 3551: RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc3551.txt): 5.1. RTP Fixed Header Fields

## RTP 协议拓展

### RTP over RTSP

RTP over RTSP 并不是在 RTP 协议文档（RFC 3550）中定义，而是在 RTSP 协议文档（RFC 2326）中定义的二进制数据交织传输方式拓展而来。

#### 1. 设置 RTP over RTSP

首次要确保 RTSP 协议交互使用的 TCP 连接，并且 RTSP 客户端需要在 SETUP 阶段请求使用 TCP 连接传输二进制数据。SETUP 命令中应该包括如下格式：

    Transport: RTP/AVP/TCP;interleaved=0-1

上述 Transport 将告诉服务端使用 TCP 协议发送媒体数据（**RTP/AVP/TCP**），并且使用信道 0 和 1 对流数据以及控制信息进行交织。详细说来，使用偶数信道作为数据传输信道，使用奇数信道作为控制信道（数据信道 + 1）。所以，如果你设定数据信道为 0 ，那控制信道应该是 0 + 1 = 1。

    C->S: SETUP rtsp://foo.com/bar.file RTSP/1.0
          CSeq: 2
          Transport: RTP/AVP/TCP;interleaved=0-1

    S->C: RTSP/1.0 200 OK
          CSeq: 2
          Date: 05 Jun 1997 18:57:18 GMT
          Transport: RTP/AVP/TCP;interleaved=0-1

#### 2. RTP over RTSP 的数据传输

> [RFC 2326: Real Time Streaming Protocol (RTSP)](https://www.rfc-editor.org/rfc/rfc2326.txt) : 10.12 Embedded (Interleaved) Binary Data

PLAY 之后，RTP 数据将通过用来发送 RTSP 命令的 TCP Socket 进行发送。RTP 数据将以如下格式进行封装：

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |    0x24($)    |      CID      |     embedded data length      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    :        ...       data      ...        :
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

* **0x24($)**：1 字节，二进制数据交错标识符，固定使用 "$"。
* **CID(channel identifier)**：1 字节，交错参数，携带 RTP 数据时用来指示信道。
    > 一般来说 CID = 0 代表视频 RTP 数据，CID = 1 代表视频 RTCP 数据  
    > CID = 2 代表音频 RTP 数据，CID = 3 代表音频 RTCP 数据
* **embedded data length**：2 个字节，用来指示插入的二进制数据长度。
* **data**：二进制数据包，为 RTP 包，长度与 embedded data length 指示的数据长度一致。

## 参考资料

* [1] [RFC 2326: Real Time Streaming Protocol (RTSP)](https://www.rfc-editor.org/rfc/rfc2326.txt)
* [2] [RFC 3550: RTP: A Transport Protocol for Real-Time Applications](https://www.rfc-editor.org/rfc/rfc3550.txt)
* [3] [DoubleLi 的博客园：《RTSP - RTP over TCP》](https://www.cnblogs.com/lidabo/p/4483497.html)
* [4] [RFC 2326: Real Time Streaming Protocol (RTSP)](https://www.rfc-editor.org/rfc/rfc2326.txt)
* [5] [RFC 3551: RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc3551.txt)
* [6] [RFC 6184: RTP Payload Format for H.264 Video](https://www.rfc-editor.org/rfc/rfc6184.txt)
* [7] [RFC 3550: RTP: A Transport Protocol for Real-Time Applications](https://www.rfc-editor.org/rfc/rfc3550.txt)
