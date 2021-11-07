---
title: 流媒体传输 - RTP 荷载 H264
categories:
  - 流媒体传输
tags:
  - RTP
abbrlink: fc237a45
date: 2020-08-12 15:53:17
---

## H264 码流结构

H264 码流是由很多 NAL Unit 组成，所有 NAL Unit 均存在一个八位数据的 NAL Unit Header ，这八位数据也充当此 RTP 有效负载格式的有效负载头。一个 NAL Unit Header 的语法如下:

        +---------------+
        |0|1|2|3|4|5|6|7|
        +-+-+-+-+-+-+-+-+
        |F|NRI|  Type   |
        +---------------+

<!-- more -->

- F: 1bit forbidden_zero_bit，在 H.264 规范中规定了这一位必须为 0。
- NRI: 2bit nal_ref_idc，取 00 ~ 11, 似乎指示这个 NALU 的重要性, 如 00 的 NALU 解码器可以丢弃它而不影响图像的回放. 不过一般情况下不太关心这个属性。
- Type: 5bit 等于 nal_unit_type，标识着这个 NAL Unit 的 Type，其类型如下：

        Table 1.  Summary of NAL unit types and the corresponding packet types
    
            NAL Unit  Packet    Packet Type Name               Section
            Type      Type
            -------------------------------------------------------------
            0        reserved                                     -
            1-23     NAL unit  Single NAL unit packet             5.6
            24       STAP-A    Single-time aggregation packet     5.7.1
            25       STAP-B    Single-time aggregation packet     5.7.1
            26       MTAP16    Multi-time aggregation packet      5.7.2
            27       MTAP24    Multi-time aggregation packet      5.7.2
            28       FU-A      Fragmentation unit                 5.8
            29       FU-B      Fragmentation unit                 5.8
            30-31    reserved

## H264 码流打包

[RFC 6184 Section 5.2](https://www.rfc-editor.org/rfc/rfc6184.txt) 中指定了 3 种打包方式：

- 单 NAL 单元模式（Single NAL Unit Packet）: 仅包含单个 NAL 单元的有效载荷。
- 组合封包模式（Aggregation Packet）：用于聚合多个 NAL 单元的分组类型成为单个 RTP 有效负载。
- 分片封包模式（Fragmentation Unit）：用于将单个 NAL 单元分段成多个 RTP 数据包。

### 单 NAL 单元模式

一个 RTP 包仅由一个完整的 NALU 组成. 这种情况下 RTP 的 NAL 头类型字段和原始的 H.264 的 NALU 头类型字段是一样的。

对于 NALU 的长度小于 MTU 大小的包, 一般采用单一 NAL 单元模式。一个原始的 H.264 NALU 单元常由 [Start Code] [NALU Header] [NALU Payload] 三部分组成, 其中 Start Code 用于标示这是一个 NALU 单元的开始, 必须是 "00 00 00 01" 或 "00 00 01", NALU 头仅一个字节, 其后都是 NALU 单元内容，打包时去除 "00 00 01" 或 "00 00 00 01" 的开始码, 把其他数据封包为 RTP 包即可。

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |F|NRI|  Type   |                                               |
    +-+-+-+-+-+-+-+-+                                               |
    |                                                               |
    |               Bytes 2..n of a single NAL unit                 |
    |                                                               |
    |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                               :...OPTIONAL RTP padding        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    
        Figure 2.  RTP payload format for single NAL unit packet

### 组合封包模式

可能是由多个 NAL 单元组成一个 RTP 包。分别有 4 种组合方式: STAP-A, STAP-B, MTAP16, MTAP24，类型值分别是 24, 25, 26 以及 27。

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |F|NRI|  Type   |                                               |
    +-+-+-+-+-+-+-+-+                                               |
    |                                                               |
    |             one or more aggregation units                     |
    |                                                               |
    |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                               :...OPTIONAL RTP padding        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    
        Figure 3.  RTP payload format for aggregation packets

MTAP 和 STAP 共享以下打包规则：

- RTP 时间戳必须设置为所有要组合的 NAL 单元中 NALU-times 的最早时间。
- 必须将 NAL 单元类型八位字节的类型字段设置为适当的值，如表 4 所示。
- 如果聚合的 NAL 的所有 F 位均为零，则必须置零 F 位；否则，必须设置它。
- NRI 的值必须是所有在聚合数据包中 NAL 单元的最大值。

             Table 4.  Type field for STAPs and MTAPs

        Type   Packet    Timestamp offset   DON-related fields
                        field length       (DON, DONB, DOND)
                        (in bits)          present
        --------------------------------------------------------
        24     STAP-A       0                 no
        25     STAP-B       0                 yes
        26     MTAP16      16                 yes
        27     MTAP24      24                 yes

### 分片封包模式

用于把一个 NALU 单元封装成多个 RTP 包. 存在两种类型 FU-A 和 FU-B. 类型值分别是 28 和 29。
这种载荷类型允许将 NAL 单元分为多个 RTP 包。该封包模式具有以下优点：

- 有效负载格式能够在 IPv4 网络传输大于 64 KB 的 NAL 单元，特别是在高清晰度格式中（每张图片的切片数量有限，导致每张图片的 NAL 单位数上限，可能会导致 NAL 单位数变大）
- 分段机制允许分段单个 NAL 单元并应用通用前向纠错。

#### FU-A

下图给出了 FU-As 的 RTP 有效载荷格式：

        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        | FU indicator  |   FU header   |                               |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
        |                                                               |
        |                         FU payload                            |
        |                                                               |
        |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                               :...OPTIONAL RTP padding        |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    
                Figure 14.  RTP payload format for FU-A

* FU indicator 格式与 NAL Header 一致:

        +---------------+
        |0|1|2|3|4|5|6|7|
        +-+-+-+-+-+-+-+-+
        |F|NRI|  Type   |
        +---------------+

Type 字段的值等于 28 和 29 分别标识该 RTP 包的封包模式为 FU-A 和 FU-B。

* FU header 格式如下:

        +---------------+
        |0|1|2|3|4|5|6|7|
        +-+-+-+-+-+-+-+-+
        |S|E|R|  Type   |
        +---------------+

    - S: 1 bit 开始位
        * 当设置成 1 时，指示跟随的 FU 荷载是分片 NAL 单元的开始。
        * 当设置成 0 时，指示跟随的 FU 荷载不是分片 NAL 单元荷载的开始。
    - E: 1 bit 结束位
        * 当设置成 1 时，指示跟随的 FU 荷载是分片 NAL 单元的结束。
        * 当设置成 1 时，指示跟随的 FU 荷载不是分片 NAL 单元的结束。
    - R: 1 bit 保留位
        * 保留位必须设置为 0。
    - Type: 5 bits
        * NAL 单元荷载类型定义见下表：
        ![NAL 单元荷载类型定义](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200814220249.jpg)

以两段 Wirshark 抓包的码流为例：

1. FU - Start

        0000   7c 85 b8 00 00 1d dc 40 00 02 ff ea 14 34 0c ab   |......@.....4..
        0010   5a 02 6f 1a 0f d5 5c f3 a9 df 6f d6 6b a8 25 1f   Z.o...\...o.k.%.
    
        ===================== 转化 第一个字节 7c 第二个字节 85 =====================
        0111 1100  F=0 NRI=11 Type=28 说明是 FU-A
        1000 0101  S=1 E=0 R=0 Type=5 非参考帧
    
        ============================= Wirshark 解析 =============================
        FU header：
          1... .... = Start bit: the first packet of FU-A picture
          .0.. .... = End bit: Not the last packet of FU-A picture
          ..0. .... = Forbidden bit: 0
          ...0 0101 = Nal_unit_type: Coded slice of an IDR picture (5)

1. FU - End

        0000   7c 45 c0                                          |E.
        
        ============== 转化 第一个字节 7c 第二个字节 45 ================
    0111 1100  F=0 NRI=11 Type=28
        0100 0101  S=0 E=1 R=0 Type=5

        ======================= Wirshark 解析 =======================
        FU header：
          0... .... = Start bit: Not the first packet of FU-A picture
          .1.. .... = End bit: the last packet of FU-A picture
          ..0. .... = Forbidden bit: 0
          ...0 0101 = Nal_unit_type: Coded slice of an IDR picture (5)

#### FU-B

FU-B 的封包格式仅仅是在 FU header 后面多了 2 个字节的 DON 解码顺序号，它允许 NAL 单元的传输顺序和 NAL 单元的解码顺序不同 ，用来 指示 NAL 单元的解码顺序。

        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        | FU indicator  |   FU header   |               DON             |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
        |                                                               |
        |                         FU payload                            |
        |                                                               |
        |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                               :...OPTIONAL RTP padding        |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    
                 Figure 15.  RTP payload format for FU-B

## 参考资料

* [1] [RFC 6184: RTP Payload Format for H.264 Video](https://www.rfc-editor.org/rfc/rfc6184.txt)
* [2] [《基于 web 浏览的视频监控与回溯系统的研究与设计》](http://cdmd.cnki.com.cn/Article/CDMD-10293-1016294288.htm)
* [3] [RFC 3550: RTP: A Transport Protocol for Real-Time Applications](https://www.rfc-editor.org/rfc/rfc3550.txt)
* [4] [RFC 2326: Real Time Streaming Protocol (RTSP)](https://www.rfc-editor.org/rfc/rfc2326.txt)
* [5] [RFC 3551: RTP Profile for Audio and Video Conferences with Minimal Control](https://www.rfc-editor.org/rfc/rfc3551.txt)
* [6] [RFC 3550: RTP: A Transport Protocol for Real-Time Applications](https://www.rfc-editor.org/rfc/rfc3550.txt)
