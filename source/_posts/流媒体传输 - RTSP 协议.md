---
title: 流媒体传输 - RTSP 协议
categories:
  - 流媒体传输
tags:
  - RTSP
abbrlink: cf84018a
date: 2020-07-27 13:33:00
---

<!-- toc -->

## 概述

### 协议简介

#### RTSP

RTSP(Real-Time Stream Protocol) 实时流传输协议是一种基于文本的应用层协议，常被用于 **建立的控制媒体流的传输**，该协议用于 **C/S 模型**, 是一个 **基于文本** 的协议, 用于在客户端和服务器端建立和协商实时流会话。

<!-- more -->

#### RTP

RTP(Real-time Transport Potocol) 实时传输协议，用于 **实时数据的传输**。

> 详见：{% post_link "流媒体传输 - RTP 协议" %}

#### RTCP

RTCP(Real-time Transport Control Protocol) 实时传输控制协议， `RTCP` 为 `RTP` 数据流提供 **信道外控制**，RTCP 的主要功能是保证服务质量，**为 RTP 提供服务质量反馈**。

> 详见：{% post_link "流媒体传输 - RTP 协议" %}

### 传输渠道

| 协议名称 | 协议文档 | 传输层协议 | 功能 |
| - | - | - | - |
| RTSP | [RFC 2326](https://www.rfc-editor.org/rfc/rfc2326.pdf)<br>[RFC 7836](https://tools.ietf.org/pdf/rfc7826.pdf) | TCP/UDP | 控制媒体流的传输 |
| RTP | [RFC 3550](https://www.rfc-editor.org/rfc/rfc3550.pdf)<br>[RFC 3551](https://www.rfc-editor.org/rfc/rfc3551.pdf)<br>[RFC 6184](https://www.rfc-editor.org/rfc/rfc6184.pdf) | UDP/TCP | 媒体流的传输 |
| RTCP | [RFC 3550](https://www.rfc-editor.org/rfc/rfc3550.pdf) | UDP/TCP | 传输质量反馈 |

## RTSP 协议

### RTSP URL

    rtsp_URL = "rtsp://" host [":" port] [ abs_path ]
    host: 有效的域名或 IP 地址
    port: 端口号，缺省为 554，若为缺省可不填写，否则必须写明
    例如：rtsp://media.example.com:554/twister/audiotrack

以海康摄像机为例，其 RTSP URL 格式为：

    rtsp://[username]:[password]@[ip]:[port]/[channel]/[subtype]/av_stream
    例如：
    rtsp://admin:12345@192.168.1.67:554/h264/ch1/main/av_stream
    rtsp://admin:12345@192.168.1.67/mpeg4/ch1/sub/av_stream

### RTSP 报文

RTSP 是一种基于文本的协议，用 CRLF(回车换行) 作为每一行的结束符，其好处是，在使用过程中可以方便地增加自定义参数，也方便抓包分析。从消息传送方向上来分，RTSP 的报文有两类：请求报文和响应报文。请求报文是指从客户端向服务器发送的请求 (也有少量从服务器向客户端发送的请求)，响应报文是指从服务器到客户端的回应。

RTSP 请求报文的常用方法：

| 方法 | 方向 | 对象 | 是否必要 | 作用 |
| - | - | - | - | - |
| DESCRIBE | C->S | P, S | 推荐 | 得到会话描述信息 |
| ANNOUNCE | C->S, S->C | P, S | 可选 |
| GET_PARAMETER | C->S, S->C | P, S | 可选 |
| OPTIONS | C->S, S->C | P, S | 必须 (S->C: 可选) | 获得服务器提供的可用方法 |
| PAUSE | C->S | P, S | 推荐 | 客户端发起暂停播放请求 |
| PLAY | C->S | P, S | 必须 | 客户端发起播放请求 |
| RECORD | C->S | P, S | 可选 |
| REDIRECT | S->C | P, S | 可选 |
| SETUP | C->S | S | 必须 | 客户端请求建立会话 |
| SET_PARAMETER | C->S, S->C | P, S | 可选 |
| TEARDOWN | C->S | P, S | 必须 | 客户端发起关闭会话 |

通过 `VLC` 播放 `RTSP` 网络流，经抓包得以下内容：

    OPTIONS rtsp://192.168.199.152:554/live/test RTSP/1.0
    CSeq: 2
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)

    RTSP/1.0 200 OK
    CSeq: 2
    Date: Mon, Jul 27 2020 15:32:38 GMT
    Public: OPTIONS, DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE, ANNOUNCE, RECORD, SET_PARAMETER, GET_PARAMETER
    Server: ZLMediaKit-5.0(build in Jul 12 2020 14:02:13)

    DESCRIBE rtsp://192.168.199.152:554/live/test RTSP/1.0
    CSeq: 3
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Accept: application/sdp

    RTSP/1.0 200 OK
    Content-Base: rtsp://192.168.199.152:554/live/test/
    Content-Length: 544
    Content-Type: application/sdp
    CSeq: 3
    Date: Mon, Jul 27 2020 15:32:38 GMT
    Server: ZLMediaKit-5.0(build in Jul 12 2020 14:02:13)
    Session: hEu0JzMKcfK2
    x-Accept-Dynamic-Rate: 1
    x-Accept-Retransmit: our-retransmit

    v=0
    o=- 0 0 IN IP4 127.0.0.1
    c=IN IP4 127.0.0.1
    t=0 0
    s=Streamed by ZLMediaKit-5.0(build in Jul 12 2020 14:01:57)
    a=tool:libavformat 58.29.100
    m=video 0 RTP/AVP 96
    a=fmtp:96 packetization-mode=1; sprop-parameter-sets=Z2QAHqzZQNg95f/wFAAUEQAAAwABAAADADIPFi2W,aOvjyyLA; profile-level-id=64001E
    a=rtpmap:96 H264/90000
    a=control:streamid=0
    m=audio 0 RTP/AVP 97
    b=AS:128
    a=fmtp:97 profile-level-id=1;mode=AAC-hbr;sizelength=13;indexlength=3;indexdeltalength=3; config=121056E500
    a=rtpmap:97 MPEG4-GENERIC/44100/2
    a=control:streamid=1
    SETUP rtsp://192.168.199.152:554/live/test/streamid=0 RTSP/1.0
    CSeq: 4
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Transport: RTP/AVP;unicast;client_port=60836-60837

    RTSP/1.0 200 OK
    CSeq: 4
    Date: Mon, Jul 27 2020 15:32:38 GMT
    Server: ZLMediaKit-5.0(build in Jul 12 2020 14:02:13)
    Session: hEu0JzMKcfK2
    Transport: RTP/AVP/UDP;unicast;client_port=60836-60837;server_port=41070-41071;ssrc=50E36E3B

    SETUP rtsp://192.168.199.152:554/live/test/streamid=1 RTSP/1.0
    CSeq: 5
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Transport: RTP/AVP;unicast;client_port=60838-60839
    Session: hEu0JzMKcfK2

    RTSP/1.0 200 OK
    CSeq: 5
    Date: Mon, Jul 27 2020 15:32:38 GMT
    Server: ZLMediaKit-5.0(build in Jul 12 2020 14:02:13)
    Session: hEu0JzMKcfK2
    Transport: RTP/AVP/UDP;unicast;client_port=60838-60839;server_port=58452-58453;ssrc=3463B21F

    PLAY rtsp://192.168.199.152:554/live/test/ RTSP/1.0
    CSeq: 6
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Session: hEu0JzMKcfK2
    Range: npt=0.000-

    RTSP/1.0 200 OK
    CSeq: 6
    Date: Mon, Jul 27 2020 15:32:38 GMT
    Range: npt=42535.41
    RTP-Info: url=rtsp://192.168.199.152:554/live/test/streamid=0;seq=3889;rtptime=-466780396,url=rtsp://192.168.199.152:554/live/test/streamid=1;seq=1155;rtptime=-179511072
    Server: ZLMediaKit-5.0(build in Jul 12 2020 14:02:13)
    Session: hEu0JzMKcfK2

    TEARDOWN rtsp://192.168.199.152:554/live/test/ RTSP/1.0
    CSeq: 7
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)
    Session: hEu0JzMKcfK2

    RTSP/1.0 200 OK
    CSeq: 7
    Date: Mon, Jul 27 2020 15:32:41 GMT
    Server: ZLMediaKit-5.0(build in Jul 12 2020 14:02:13)
    Session: hEu0JzMKcfK2

从上述抓包中看出，VLC 在播放 RTSP 网络流时，客户端与服务端经过了 6 次交互：

| 序号 | 方向 | 方法 | 消息内容 |
| - | - | - | - |
| 1 | C->S | OPTIONS | Client 询问 Server 有哪些方法可用 |
| 1 | S->C | OPTIONS | Server 回应所有可用的方法 |
| 2 | C->S | DESCRIBE | Client 请求得到 Server 提供的媒体初始化描述信息 |
| 2 | S->C | DESCRIBE | Server 回应媒体初始化信息，主要是 SDP （会话描述协议） |
| 3 | C->S | SETUP | 设置视频会话属性以及传输模式，请求建立会话 |
| 3 | S->C | SETUP | Server 建立会话，返回会话标识以及会话相关信息 |
| 4 | C->S | SETUP | 设置音频会话属性以及传输模式，请求建立会话  |
| 4 | S->C | SETUP | Server 建立会话，返回会话标识以及会话相关信息 |
| 5 | C->S | PLAY | Client 请求播放 |
| 5 | S->C | PLAY | Server 回应播放请求 |
| 6 | C->S | TEARDOWN | Client 请求关闭会话 |
| 6 | S->C | TEARDOWN | Server 回应关闭会话请求 |

下面我们一步一步了解 RTSP 会话建立流程：

### OPTIONS

OPTIONS 请求可以在任何时间被发出，而且不会影响到 Server 的状态，例如：

     C->S:  OPTIONS * RTSP/1.0
            CSeq: 1
            Require: implicit-play
            Proxy-Require: gzipped-messages

     S->C:  RTSP/1.0 200 OK
            CSeq: 1
            Public: DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE

### DESCRIBE

客户端向服务器请求媒体资源描述，服务器端通过 SDP(Session Description Protocol) 格式回应客户端的请求。资源描述中会列出所请求媒体的媒体流及其相关信息，典型情况下，音频和视频分别作为一个媒体流传输。例如：

     C->S: DESCRIBE rtsp://server.example.com/fizzle/foo RTSP/1.0
           CSeq: 312
           Accept: application/sdp, application/rtsl, application/mheg

     S->C: RTSP/1.0 200 OK
           CSeq: 312
           Date: 23 Jan 1997 15:35:06 GMT
           Content-Type: application/sdp
           Content-Length: 376

           v=0
           o=mhandley 2890844526 2890842807 IN IP4 126.16.64.4
           s=SDP Seminar
           i=A Seminar on the session description protocol
           u=http://www.cs.ucl.ac.uk/staff/M.Handley/sdp.03.ps
           e=mjh@isi.edu (Mark Handley)
           c=IN IP4 224.2.17.12/127
           t=2873397496 2873404696
           a=recvonly
           m=audio 3456 RTP/AVP 0
           m=video 2232 RTP/AVP 31
           m=whiteboard 32416 UDP WB
           a=orient:portrait

> 媒体流及其相关信息由 SDP(Session Description Protocol) 格式携带，关于 SDP 的详细说明，参见拓展阅读 5

### SETUP

SETUP 请求确定了具体的媒体流如何传输，该请求必须在 PLAY 请求之前发送。SETUP 请求包含媒体流的 URL 和客户端用于接收 RTP 数据 (audio or video) 的端口以及接收 RTCP 数据 (meta information) 的端口。服务器端的回复通常包含客户端请求参数的确认，并会补充缺失的部分，比如服务器选择的发送端口。每一个媒体流在发送 PLAY 请求之前，都要首先通过 SETUP 请求来进行相应的配置。

    C->S: SETUP rtsp://example.com/foo/bar/baz.rm RTSP/1.0
          CSeq: 302
          Transport: RTP/AVP;unicast;client_port=4588-4589

    S->C: RTSP/1.0 200 OK
          CSeq: 302
          Date: 23 Jan 1997 15:35:06 GMT
          Session: 47112344
          Transport: RTP/AVP;unicast;client_port=4588-4589;server_port=6256-6257

### PLAY

客户端通过 PLAY 请求来播放一个或全部媒体流，PLAY 请求可以发送一次或多次，发送一次时，URL 为包含所有媒体流的地址，发送多次时，每一次请求携带的 URL 只包含一个相应的媒体流。PLAY 请求中可指定播放的 range，若未指定，则从媒体流的开始播放到结束，如果媒体流在播放过程中被暂停，则可在暂停处重新启动流的播放。

    C->S: PLAY rtsp://example.com/media.mp4 RTSP/1.0
          CSeq: 4
          Range: npt=5-20
          Session: 12345678

    S->C: RTSP/1.0 200 OK
          CSeq: 4
          Session: 12345678
          RTP-Info: url=rtsp://example.com/media.mp4/streamid=0;seq=9810092;rtptime=3450012

> Server 处理 client 发来的 PLAY 请求后，就会开始向 client 发送媒体数据，一般采用 RTP 协议进行发送，关于 RTP 协议的相关说明，参见拓展阅读 6

### TEARDOWN

结束会话请求，该请求会停止所有媒体流，并释放服务器上的相关会话数据。

    C->S: TEARDOWN rtsp://example.com/media.mp4 RTSP/1.0
          CSeq: 8
          Session: 12345678

    S->C: RTSP/1.0 200 OK
          CSeq: 8

## 参考资料

* [1] [ZLMediaKit](https://github.com/xia-chu/ZLMediaKit) （作为 RTSP Server）
* [2] [VLC](https://www.videolan.org/) （作为 RTSP Client）
* [3] [yooooooo 的博客园：《网络流媒体协议之——RTSP 协议》](https://www.cnblogs.com/linhaostudy/p/11140823.html)
* [4] [RFC 2326: Real Time Streaming Protocol (RTSP)](https://www.rfc-editor.org/rfc/rfc2326.txt)
* [5] [RFC 3551: RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc3551.txt)
* [6] [RFC 6184: RTP Payload Format for H.264 Video](https://www.rfc-editor.org/rfc/rfc6184.txt)
* [7] [RFC 3550: RTP: A Transport Protocol for Real-Time Applications](https://www.rfc-editor.org/rfc/rfc3550.txt)
