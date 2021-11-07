---
title: 流媒体传输 - RTSP Over HTTP
categories:
  - 流媒体传输
tags:
  - RTSP
abbrlink: 65370ad
date: 2020-09-14 16:07:12
---
RTSP 的标准端口是 554，但是由于各种不同的防火墙等安全策略配置的原因，客户端在访问 554 端口时可能存在限制，从而无法正常传输 RTSP 报文。
但是 HTTP 端口（80 端口）是普遍开放的，于是就有了让 RTSP 报文通过 80 端口透传的想法，即 RTSP Over HTTP。

<!-- more -->

## 协议介绍

RTSP Over HTTP 的关键在于：让 RTSP 报文通过 HTTP 端口通信，但目前 RTSP Over HTTP 没有标准做法，苹果公司出了一份非正式文档公开在外，并且也被 Live555 等支持

## 基础知识

### RTSP 和 HTTP

RTSP(Real Time Streaming Protocol，实时流传输协议) 和 HTTP(HyperText Transfer Protocol，超文本传输协议) 的共同点如下：

- 两者均为应用层协议
- 两者均为工作于客户端 - 服务端架构

两者区别如下：

- HTTP 协议是无连接（HTTP/1.1 版本之后支持长连接），而 RTSP 为面向连接协议
    > 无连接的含义是限制每次连接只处理一个请求。服务器处理完客户的请求，并收到客户的应答后，即断开连接。
- HTTP 协议是无状态协议，而 RTSP 为有状态协议

## 协议交互

Live555 的具体做法如下

首先客户端开启 2 个 socket 链接服务器 HTTP 端口，我们称这 2 个 socket 分别为 "数据 socket" 和 "命令 socket"。

1. 客户端通过 "数据 socket" 发送 HTTP GET 命令，请求 RTSP 链接。
2. 服务器通过 "数据 socket" 响应 HTTP GET 命令，并回复成功 / 失败。
3. 客户端创建 "命令 socket"，并通过 "命令 socket" 发送 HTTP POST 命令，建立 RTSP 会话。

    至此，HTTP 的辅助功能完成，服务器不返回客户端的 HTTP POST 命令。接下来是 RTSP 在 HTTP 端口上的标准流程，但是需要通过 2 个 socket 协同完成，"命令 socket" 只负责发送，"数据 socket" 只负责接受。

4. 客户端通过 "命令 socket" 发送 RTSP 命令（BASE64 编码加密）。
5. 服务器通过 "数据 socket" 响应 RTSP 命令（明文）。
6. 重复 Step4-Step5，直到客户端发送 RTSP PLAY 命令，服务器响应 RTSP PLAY 命令。
7. 服务器通过 数据 socket" 向客户端传输音视频数据

    数据交互...

8. 客户端通过 "命令 socket" 发送 RTSP TEARDOWN 命令（BASE64 编码加密）
9. 服务器通过 "数据 socket" 响应 RTSP TEARDOWN 命令（明文）。
10. 关闭 2 个 socket。

## 交互示例

通过海康的 IPC 和 海康播放器 VSPlayer 进行抓包，得到数据交互过程如下：

> 由于有两个连接之间进行交互 直接通过 Wirshark 观察数据不够清晰，这里对每次交互的数据方向和通道进行了注释

1. Data Socket, C -> S

        GET /ch1/main/av_stream HTTP/1.0
        CSeq: 1
        User-Agent: NKPlayer-VSPlayer1.0
        x-sessioncookie: 6521ade129b5b5e14e8eacd
        Accept: application/x-rtsp-tunnelled
        Pragma: no-cache
        Cache-Control: no-cache

2. Data Socket, S -> C

        HTTP/1.1 200 OK
        Connection: close
        Content-Type: application/x-rtsp-tunnelled

3. Command Socket, C -> S

        POST /ch1/main/av_stream HTTP/1.0
        CSeq: 1
        User-Agent: NKPlayer-VSPlayer1.0
        x-sessioncookie: 6521ade129b5b5e14e8eacd
        Content-Type: application/x-rtsp-tunnelled
        Pragma: no-cache
        Cache-Control: no-cache
        Content-Length: 32767
        Expires: Sun, 9 Jan 1972 00:00:00 GMT

    > 至此，HTTP 的辅助功能完成，服务器不返回客户端的 HTTP POST 命令。接下来是 RTSP 在 HTTP 端口上的标准流程，但是需要通过 2 个 socket 协同完成，"命令 socket" 只负责发送，"数据 socket" 只负责接受。

4. Command Socket, C -> S

        T1BUSU9OUyBydHNwOi8vMTAuMTkyLjQ0Ljk3OjU1NC9jaDEvbWFpbi9hdl9zdHJlYW0gUlRTUC8xLjANCkNTZXE6IDINClVzZXItQWdlbnQ6IE5LUGxheWVyLVZTUGxheWVyMS4wDQoNCg==
        -------------------------------------------------------------------
        OPTIONS rtsp://10.192.44.97:554/ch1/main/av_stream RTSP/1.0
        CSeq: 2
        User-Agent: NKPlayer-VSPlayer1.0

5. Data Socket, S -> C

        RTSP/1.0 200 OK
        CSeq: 2
        Public: OPTIONS, DESCRIBE, PLAY, PAUSE, SETUP, TEARDOWN, SET_PARAMETER
        Date:  Mon, Sep 21 2020 20:08:41 GMT

6. Command Socket, C -> S

        REVTQ1JJQkUgcnRzcDovLzEwLjE5Mi40NC45Nzo1NTQvY2gxL21haW4vYXZfc3RyZWFtIFJUU1AvMS4wDQpDU2VxOiAzDQpVc2VyLUFnZW50OiBOS1BsYXllci1WU1BsYXllcjEuMA0KQWNjZXB0OiBhcHBsaWNhdGlvbi9zZHANCg0K
        -------------------------------------------------------------------
        DESCRIBE rtsp://10.192.44.97:554/ch1/main/av_stream RTSP/1.0
        CSeq: 3
        User-Agent: NKPlayer-VSPlayer1.0
        Accept: application/sdp

7. Data Socket, S -> C

        RTSP/1.0 401 Unauthorized
        CSeq: 3
        WWW-Authenticate: Digest realm="IP Camera(E6990)", nonce="921e0ab66f8f6763ef05ecb06c4c86a0", stale="FALSE"
        WWW-Authenticate: Basic realm="IP Camera(E6990)"
        Date:  Mon, Sep 21 2020 20:08:41 GMT

8. Command Socket, C -> S

        REVTQ1JJQkUgcnRzcDovLzEwLjE5Mi40NC45Nzo1NTQvY2gxL21haW4vYXZfc3RyZWFtIFJUU1AvMS4wDQpDU2VxOiA0DQpBdXRob3JpemF0aW9uOiBEaWdlc3QgdXNlcm5hbWU9ImFkbWluIiwgcmVhbG09IklQIENhbWVyYShFNjk5MCkiLCBub25jZT0iOTIxZTBhYjY2ZjhmNjc2M2VmMDVlY2IwNmM0Yzg2YTAiLCB1cmk9InJ0c3A6Ly8xMC4xOTIuNDQuOTc6NTU0L2NoMS9tYWluL2F2X3N0cmVhbSIsIHJlc3BvbnNlPSIwNjk0ZjhkMmNlODE3NzE5OTIxOTk1MzJkNDdiNzNhZCINClVzZXItQWdlbnQ6IE5LUGxheWVyLVZTUGxheWVyMS4wDQpBY2NlcHQ6IGFwcGxpY2F0aW9uL3NkcA0KDQo=
        -------------------------------------------------------------------
        DESCRIBE rtsp://10.192.44.97:554/ch1/main/av_stream RTSP/1.0
        CSeq: 4
        Authorization: Digest username="admin", realm="IP Camera(E6990)", nonce="921e0ab66f8f6763ef05ecb06c4c86a0", uri="rtsp://10.192.44.97:554/ch1/main/av_stream", response="0694f8d2ce81771992199532d47b73ad"
        User-Agent: NKPlayer-VSPlayer1.0
        Accept: application/sdp

9. Data Socket, S -> C
        RTSP/1.0 200 OK
        CSeq: 4
        Content-Type: application/sdp
        Content-Length: 633

        v=0
        o=- 1600718921878550 1600718921878550 IN IP6 6d69:6e22:3a09:302c:a09:909:909:2240
        s=Media Presentation
        e=NONE
        b=AS:5100
        t=0 0
        a=control:*
        m=video 0 RTP/AVP 96
        c=IN IP4 0.0.0.0
        b=AS:5000
        a=recvonly
        a=x-dimensions:2560,1440
        a=control:trackID=1
        a=rtpmap:96 H264/90000
        a=fmtp:96 profile-level-id=420029; packetization-mode=1; sprop-parameter-sets=Z00AMpY1QFABa03AQEBQAABwgAAV+QBA,aO48gA==
        m=audio 0 RTP/AVP 8
        c=IN IP4 0.0.0.0
        b=AS:50
        a=recvonly
        a=control:trackID=2
        a=rtpmap:8 PCMA/8000
        a=Media_header:MEDIAINFO=494D4B48010200000400000111710110401F000000FA000000000000000000000000000000000000;
        a=appversion:1.0

10. Command Socket, C -> S

        U0VUVVAgcnRzcDovLzEwLjE5Mi40NC45Nzo1NTQvY2gxL21haW4vYXZfc3RyZWFtL3RyYWNrSUQ9MSBSVFNQLzEuMA0KQ1NlcTogNQ0KQXV0aG9yaXphdGlvbjogRGlnZXN0IHVzZXJuYW1lPSJhZG1pbiIsIHJlYWxtPSJJUCBDYW1lcmEoRTY5OTApIiwgbm9uY2U9IjkyMWUwYWI2NmY4ZjY3NjNlZjA1ZWNiMDZjNGM4NmEwIiwgdXJpPSJydHNwOi8vMTAuMTkyLjQ0Ljk3OjU1NC9jaDEvbWFpbi9hdl9zdHJlYW0iLCByZXNwb25zZT0iNmRmZTdiMjhhMmZmZjliMTYxYmFjNWRkYWQxMjg5ZTQiDQpVc2VyLUFnZW50OiBOS1BsYXllci1WU1BsYXllcjEuMA0KVHJhbnNwb3J0OiBSVFAvQVZQL1RDUDt1bmljYXN0O2ludGVybGVhdmVkPTAtMQ0KDQo=
        -------------------------------------------------------------------
        SETUP rtsp://10.192.44.97:554/ch1/main/av_stream/trackID=1 RTSP/1.0
        CSeq: 5
        Authorization: Digest username="admin", realm="IP Camera(E6990)", nonce="921e0ab66f8f6763ef05ecb06c4c86a0", uri="rtsp://10.192.44.97:554/ch1/main/av_stream", response="6dfe7b28a2fff9b161bac5ddad1289e4"
        User-Agent: NKPlayer-VSPlayer1.0
        Transport: RTP/AVP/TCP;unicast;interleaved=0-1

11. Data Socket, S -> C

        RTSP/1.0 200 OK
        CSeq: 5
        Session:       2090545605;timeout=60
        Transport: RTP/AVP/TCP;unicast;interleaved=0-1;ssrc=11c14094;mode="play"
        Date:  Mon, Sep 21 2020 20:08:41 GMT

12. Command Socket, C -> S

        U0VUVVAgcnRzcDovLzEwLjE5Mi40NC45Nzo1NTQvY2gxL21haW4vYXZfc3RyZWFtL3RyYWNrSUQ9MiBSVFNQLzEuMA0KQ1NlcTogNg0KQXV0aG9yaXphdGlvbjogRGlnZXN0IHVzZXJuYW1lPSJhZG1pbiIsIHJlYWxtPSJJUCBDYW1lcmEoRTY5OTApIiwgbm9uY2U9IjkyMWUwYWI2NmY4ZjY3NjNlZjA1ZWNiMDZjNGM4NmEwIiwgdXJpPSJydHNwOi8vMTAuMTkyLjQ0Ljk3OjU1NC9jaDEvbWFpbi9hdl9zdHJlYW0iLCByZXNwb25zZT0iNmRmZTdiMjhhMmZmZjliMTYxYmFjNWRkYWQxMjg5ZTQiDQpVc2VyLUFnZW50OiBOS1BsYXllci1WU1BsYXllcjEuMA0KVHJhbnNwb3J0OiBSVFAvQVZQL1RDUDt1bmljYXN0O2ludGVybGVhdmVkPTItMw0KU2Vzc2lvbjogMjA5MDU0NTYwNQ0KDQo=
        -------------------------------------------------------------------
        SETUP rtsp://10.192.44.97:554/ch1/main/av_stream/trackID=2 RTSP/1.0
        CSeq: 6
        Authorization: Digest username="admin", realm="IP Camera(E6990)", nonce="921e0ab66f8f6763ef05ecb06c4c86a0", uri="rtsp://10.192.44.97:554/ch1/main/av_stream", response="6dfe7b28a2fff9b161bac5ddad1289e4"
        User-Agent: NKPlayer-VSPlayer1.0
        Transport: RTP/AVP/TCP;unicast;interleaved=2-3
        Session: 2090545605

13. Data Socket, S -> C

        RTSP/1.0 200 OK
        CSeq: 6
        Session:       2090545605;timeout=60
        Transport: RTP/AVP/TCP;unicast;interleaved=2-3;ssrc=6819e89f;mode="play"
        Date:  Mon, Sep 21 2020 20:08:41 GMT

14. Command Socket, C -> S

        UExBWSBydHNwOi8vMTAuMTkyLjQ0Ljk3OjU1NC9jaDEvbWFpbi9hdl9zdHJlYW0gUlRTUC8xLjANCkNTZXE6IDcNCkF1dGhvcml6YXRpb246IERpZ2VzdCB1c2VybmFtZT0iYWRtaW4iLCByZWFsbT0iSVAgQ2FtZXJhKEU2OTkwKSIsIG5vbmNlPSI5MjFlMGFiNjZmOGY2NzYzZWYwNWVjYjA2YzRjODZhMCIsIHVyaT0icnRzcDovLzEwLjE5Mi40NC45Nzo1NTQvY2gxL21haW4vYXZfc3RyZWFtIiwgcmVzcG9uc2U9IjkzN2VhNWIyMjUxNGViNzVjZjIxNTNiODZmOTEzMmQyIg0KVXNlci1BZ2VudDogTktQbGF5ZXItVlNQbGF5ZXIxLjANClNlc3Npb246IDIwOTA1NDU2MDUNClJhbmdlOiBucHQ9MC4wMDAtDQoNCg==
        -------------------------------------------------------------------
        PLAY rtsp://10.192.44.97:554/ch1/main/av_stream RTSP/1.0
        CSeq: 7
        Authorization: Digest username="admin", realm="IP Camera(E6990)", nonce="921e0ab66f8f6763ef05ecb06c4c86a0", uri="rtsp://10.192.44.97:554/ch1/main/av_stream", response="937ea5b22514eb75cf2153b86f9132d2"
        User-Agent: NKPlayer-VSPlayer1.0
        Session: 2090545605
        Range: npt=0.000-

15. Data Socket, S -> C

        RTSP/1.0 200 OK
        CSeq: 7
        Session:       2090545605
        RTP-Info: url=trackID=1;seq=19485,url=trackID=2;seq=5076
        Date:  Mon, Sep 21 2020 20:08:42 GMT

16. Command Socket, C -> S

        VEVBUkRPV04gcnRzcDovLzEwLjE5Mi40NC45Nzo1NTQvY2gxL21haW4vYXZfc3RyZWFtIFJUU1AvMS4wDQpDU2VxOiA4DQpBdXRob3JpemF0aW9uOiBEaWdlc3QgdXNlcm5hbWU9ImFkbWluIiwgcmVhbG09IklQIENhbWVyYShFNjk5MCkiLCBub25jZT0iOTIxZTBhYjY2ZjhmNjc2M2VmMDVlY2IwNmM0Yzg2YTAiLCB1cmk9InJ0c3A6Ly8xMC4xOTIuNDQuOTc6NTU0L2NoMS9tYWluL2F2X3N0cmVhbSIsIHJlc3BvbnNlPSJiMDQxZWY0Yzk3ZDExZTYyMjUwNmFjZjhiZTBlYmZkNyINClVzZXItQWdlbnQ6IE5LUGxheWVyLVZTUGxheWVyMS4wDQpTZXNzaW9uOiAyMDkwNTQ1NjA1DQoNCg==
        -------------------------------------------------------------------
        TEARDOWN rtsp://10.192.44.97:554/ch1/main/av_stream RTSP/1.0
        CSeq: 8
        Authorization: Digest username="admin", realm="IP Camera(E6990)", nonce="921e0ab66f8f6763ef05ecb06c4c86a0", uri="rtsp://10.192.44.97:554/ch1/main/av_stream", response="b041ef4c97d11e622506acf8be0ebfd7"
        User-Agent: NKPlayer-VSPlayer1.0
        Session: 2090545605

17. Data Socket, S -> C

        RTSP/1.0 200 OK
        CSeq: 8
        Session:       2090545605
        Date:  Mon, Sep 21 2020 20:08:42 GMT

## 参考资料

* [1] [关于 RTSP-Over-HTTP - Ansersion - 博客园](https://www.cnblogs.com/ansersion/p/7514035.html)
