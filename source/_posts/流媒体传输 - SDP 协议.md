---
title: 流媒体传输 - SDP 协议
categories: 
    - 流媒体传输
tags:
  - SDP
abbrlink: 1fa21384
date: 2020-08-29 17:01:58
---
## SDP 协议介绍

SDP 全称是 Session Description Protocol，翻译过来就是描述会话的协议。主要用于两个会话实体之间的媒体协商。

什么叫会话呢，比如一次网络电话、一次电话会议、一次视频聊天，这些都可以称之为一次会话。

那为什么要去发这个描述文本呢，主要是为了解决参与会话的各成员之间能力不对等的问题，如果参加本次通话的成员都支持高质量的通话，但是我们没有去进行协议，为了兼容性，使用的都是普通质量的通话格式，这样就很浪费资源了。所以 SDP 的作用还是很有必要的。

<!-- more -->

## SDP 协议结构

SDP 描述由许多文本行组成，文本行的格式为 `<type> = <value>`，`<type>` 是一个字母，`<value>` 是结构化的文本串，其格式依 `<type>` 而定。

    <type> = <value>

SDP 的文本信息包括：

* 会话名称和意图
* 会话持续时间
* 构成会话的媒体
* 有关接收媒体的信息

### 会话名称和意图描述

    v =  （协议版本）
    o =  （所有者 / 创建者和会话标识符）
    s =  （会话名称）
    i = *（会话信息）
    u = *（URI 描述）
    e = *（Email 地址）
    p = *（电话号码）
    c = *（连接信息 ― 如果包含在所有媒体中，则不需要该字段）
    b = *（带宽信息）

### 时间描述

    t =  （会话活动时间）
    r = *（0 或多次重复次数）

### 媒体描述

    m =  （媒体名称和传输地址）
    i = *（媒体标题）
    c = *（连接信息 — 如果包含在会话层则该字段可选）
    b = *（带宽信息）
    k = *（加密密钥）
    a = *（0 个或多个会话属性行）

### SDP 举例

```c++
//sdp 版本号，一直为 0,rfc4566 规定
o=- 7017624586836067756 2 IN IP4 127.0.0.1
// o=<username> <sess-id> <sess-version> <nettype> <addrtype> <unicast-address>
//username 如何没有使用 - 代替，7017624586836067756 是整个会话的编号，2 代表会话版本，如果在会话
// 过程中有改变编码之类的操作，重新生成 sdp 时, sess-id 不变，sess-version 加 1
s=-
// 会话名，没有的话使用 - 代替
t=0 0
// 两个值分别是会话的起始时间和结束时间，这里都是 0 代表没有限制
a=group:BUNDLE audio video data
// 需要共用一个传输通道传输的媒体，如果没有这一行，音视频，数据就会分别单独用一个 udp 端口来发送
a=msid-semantic: WMS h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C
// WMS 是 WebRTC Media Stream 简称，这一行定义了本客户端支持同时传输多个流，一个流可以包括多个 track,
// 一般定义了这个，后面 a=ssrc 这一行就会有 msid,mslabel 等属性
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 126
// m=audio 说明本会话包含音频，9 代表音频使用端口 9 来传输，但是在 webrtc 中一现在一般不使用，如果设置为 0，代表不
// 传输音频, UDP/TLS/RTP/SAVPF 是表示用户来传输音频支持的协议，udp，tls,rtp 代表使用 udp 来传输 rtp 包，并使用 tls 加密
// SAVPF 代表使用 srtcp 的反馈机制来控制通信过程, 后台 111 103 104 9 0 8 106 105 13 126 表示本会话音频支持的编码，后台几行会有详细补充说明
c=IN IP4 0.0.0.0
// 这一行表示你要用来接收或者发送音频使用的 IP 地址，webrtc 使用 ice 传输，不使用这个地址
a=rtcp:9 IN IP4 0.0.0.0
// 用来传输 rtcp 地地址和端口，webrtc 中不使用
a=ice-ufrag:khLS
a=ice-pwd:cxLzteJaJBou3DspNaPsJhlQ
// 以上两行是 ice 协商过程中的安全验证信息
a=fingerprint:sha-256 FA:14:42:3B:C7:97:1B:E8:AE:0C2:71:03:05:05:16:8F:B9:C7:98:E9:60:43:4B:5B:2C:28:EE:5C:8F3:17
// 以上这行是 dtls 协商过程中需要的认证信息
a=setup:actpass
// 以上这行代表本客户端在 dtls 协商过程中，可以做客户端也可以做服务端，参考 rfc4145 rfc4572
a=mid:audio
// 在前面 BUNDLE 这一行中用到的媒体标识
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
// 上一行指出我要在 rtp 头部中加入音量信息，参考 rfc6464
a=sendrecv
// 上一行指出我是双向通信，另外几种类型是 recvonly,sendonly,inactive
a=rtcp-mux
// 上一行指出 rtp,rtcp 包使用同一个端口来传输
// 下面几行都是对 m=audio 这一行的媒体编码补充说明，指出了编码采用的编号，采样率，声道等
a=rtpmap:111 opus/48000/2
a=rtcp-fb:111 transport-cc
// 以上这行说明 opus 编码支持使用 rtcp 来控制拥塞，参考 https://tools.ietf.org/html/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=fmtp:111 minptime=10;useinbandfec=1
// 对 opus 编码可选的补充说明, minptime 代表最小打包时长是 10ms，useinbandfec=1 代表使用 opus 编码内置 fec 特性
a=rtpmap:103 ISAC/16000
a=rtpmap:104 ISAC/32000
a=rtpmap:9 G722/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:106 CN/32000
a=rtpmap:105 CN/16000
a=rtpmap:13 CN/8000
a=rtpmap:126 telephone-event/8000
a=ssrc:18509423 cname:sTjtznXLCNH7nbRw
// cname 用来标识一个数据源，ssrc 当发生冲突时可能会发生变化，但是 cname 不会发生变化，也会出现在 rtcp 包中 SDEC 中，
// 用于音视频同步
a=ssrc:18509423 msid:h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C 15598a91-caf9-4fff-a28f-3082310b2b7a
// 以上这一行定义了 ssrc 和 WebRTC 中的 MediaStream,AudioTrack 之间的关系，msid 后面第一个属性是 stream-d, 第二个是 track-id
a=ssrc:18509423 mslabel:h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C
a=ssrc:18509423 label:15598a91-caf9-4fff-a28f-3082310b2b7a
m=video 9 UDP/TLS/RTP/SAVPF 100 101 107 116 117 96 97 99 98
// 参考上面 m=audio, 含义类似
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:khLS
a=ice-pwd:cxLzteJaJBou3DspNaPsJhlQ
a=fingerprint:sha-256 FA:14:42:3B:C7:97:1B:E8:AE:0C2:71:03:05:05:16:8F:B9:C7:98:E9:60:43:4B:5B:2C:28:EE:5C:8F3:17
a=setup:actpass
a=mid:video
a=extmap:2 urn:ietf:params:rtp-hdrext:toffset
a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 urn:3gpp:video-orientation
a=extmap:5 http://www.ietf.org/id/draft-hol ... de-cc-extensions-01
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=sendrecv
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:100 VP8/90000
a=rtcp-fb:100 ccm fir
// ccm 是 codec control using RTCP feedback message 简称，意思是支持使用 rtcp 反馈机制来实现编码控制，fir 是 Full Intra Request
// 简称，意思是接收方通知发送方发送幅完全帧过来
a=rtcp-fb:100 nack
// 支持丢包重传，参考 rfc4585
a=rtcp-fb:100 nack pli
// 支持关键帧丢包重传, 参考 rfc4585
a=rtcp-fb:100 goog-remb
// 支持使用 rtcp 包来控制发送方的码流
a=rtcp-fb:100 transport-cc
// 参考上面 opus
a=rtpmap:101 VP9/90000
a=rtcp-fb:101 ccm fir
a=rtcp-fb:101 nack
a=rtcp-fb:101 nack pli
a=rtcp-fb:101 goog-remb
a=rtcp-fb:101 transport-cc
a=rtpmap:107 H264/90000
a=rtcp-fb:107 ccm fir
a=rtcp-fb:107 nack
a=rtcp-fb:107 nack pli
a=rtcp-fb:107 goog-remb
a=rtcp-fb:107 transport-cc
a=fmtp:107 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
// h264 编码可选的附加说明
a=rtpmap:116 red/90000
// fec 冗余编码，一般如果 sdp 中有这一行的话，rtp 头部负载类型就是 116，否则就是各编码原生负责类型
a=rtpmap:117 ulpfec/90000
// 支持 ULP FEC，参考 rfc5109
a=rtpmap:96 rtx/90000
a=fmtp:96 apt=100
// 以上两行是 VP8 编码的重传包 rtp 类型
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=101
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=107
a=rtpmap:98 rtx/90000
a=fmtp:98 apt=116
a=ssrc-group:FID 3463951252 1461041037
// 在 webrtc 中，重传包和正常包 ssrc 是不同的，上一行中前一个是正常 rtp 包的 ssrc, 后一个是重传包的 ssrc
a=ssrc:3463951252 cname:sTjtznXLCNH7nbRw
a=ssrc:3463951252 msid:h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C ead4b4e9-b650-4ed5-86f8-6f5f5806346d
a=ssrc:3463951252 mslabel:h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C
a=ssrc:3463951252 label:ead4b4e9-b650-4ed5-86f8-6f5f5806346d
a=ssrc:1461041037 cname:sTjtznXLCNH7nbRw
a=ssrc:1461041037 msid:h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C ead4b4e9-b650-4ed5-86f8-6f5f5806346d
a=ssrc:1461041037 mslabel:h1aZ20mbQB0GSsq0YxLfJmiYWE9CBfGch97C
a=ssrc:1461041037 label:ead4b4e9-b650-4ed5-86f8-6f5f5806346d
m=application 9 DTLS/SCTP 5000
c=IN IP4 0.0.0.0
a=ice-ufrag:khLS
a=ice-pwd:cxLzteJaJBou3DspNaPsJhlQ
a=fingerprint:sha-256 FA:14:42:3B:C7:97:1B:E8:AE:0C2:71:03:05:05:16:8F:B9:C7:98:E9:60:43:4B:5B:2C:28:EE:5C:8F3:17
a=setup:actpass
a=mid:data
a=sctpmap:5000 webrtc-datachannel 1024
```

## 参考资料

* [1] [RFC 4566: Session Description Protocol](https://www.rfc-editor.org/rfc/rfc4566.txt)
* [2] [SDP 协议 - 简书](https://www.jianshu.com/p/94b118b8fd97)
