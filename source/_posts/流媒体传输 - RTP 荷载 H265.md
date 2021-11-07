---
title: 流媒体传输 - RTP 荷载 H265
categories:
  - 流媒体传输
tags:
  - RTP
abbrlink: 8b244ad3
date: 2020-08-18 13:48:26
---

## H265 码流结构

H265 码流和是由很多 NAL Unit 组成，所有 NAL Unit 均存在一个 16 位数据的 NAL Unit Header ，一个 NAL Unit Header 的语法如下:

            +---------------+---------------+
            |0|1|2|3|4|5|6|7|0|1|2|3|4|5|6|7|
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |F|   Type    |  LayerId  | TID |
            +-------------+-----------------+

    Figure 1: The Structure of the H265 NAL Unit Header

<!-- more -->

- F: 1bit forbidden_zero_bit，在 H.265 规范中规定了这一位必须为 0。它的作用是在尚存 MPEG-2 系统环境中，防止产生可以解释为 MPEG-2 起始码的比特模式。
- Type: 6bit 其允许的 NAL Unit 的类型编码比 H264 多一倍，达到了 64 类，其中 32 类作用于 VCL NAL Unit，32 类作用于 non-VCL NAL Unit。
- LayerId: 6bit 用于 H265 拓展层标识符
- TID: 3bit temporal_id，表示 H265 的接入单元（AU）属于哪个时域子层，时域标识符值为 0 到 6。

## H265 码流打包

[RFC 7798 Section 4.4](https://www.rfc-editor.org/rfc/rfc7798.txt) 指定了四种不同类型的 RTP 数据包有效负载结构：

- 单 NAL 单元模式（Single NAL Unit Packet）: 仅包含单个 NAL Unit 的有效载荷。

- 组合封包模式（Aggregation Packet）：用于聚合多个 NAL Unit 的分组类型成为单个 RTP 有效负载。

- 分片封包模式（Fragmentation Unit）：用于将单个 NAL Unit 分段成多个 RTP 数据包。

- 携带 RTP 数据包的 PACI：包含有效载荷报头（与其他有效载荷报头有所不同），有效载荷报头扩展结构（PHES）和 PACI 有效载荷。

    其中常用的有两种类型：单 NAL 单元模式和分片封包模式。

### 单 NAL 单元模式

一个 RTP 包仅由一个完整的 NALU 组成. 这种情况下 RTP 的 NAL 头类型字段 PayloadHdr 和原始的 H.265 的 NALU Header 字段是一样的。

对于 NALU 的长度小于 MTU 大小的包, 一般采用单一 NAL 单元模式。一个原始的 H.264 NALU 单元常由 [Start Code] [NALU Header] [NALU Payload] 三部分组成, 其中 Start Code 用于标示这是一个 NALU 单元的开始, 必须是 "00 00 00 01" 或 "00 00 01", NALU 头仅一个字节, 其后都是 NALU 单元内容，打包时去除 "00 00 01" 或 "00 00 00 01" 的开始码, 把其他数据封包为 RTP 包即可。

    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           PayloadHdr          |      DONL (conditional)       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                  NAL unit payload data                        |
    |                                                               |
    |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                               :...OPTIONAL RTP padding        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

          Figure 3: The Structure of a Single NAL Unit Packet

### 分片封包模式

分片封包模式（FU）能将单个 NAL 单元分段成多个 RTP 数据包。

NAL 单元的片段由该 NAL 单元的整数个连续八位位组组成。 分片封包的 NAL 单元必须以升序的 RTP 序列号连续顺序发送（同一 RTP 流中的其他 RTP 数据包不得在第一个片段与最后一个片段之间发送）。

FU 绝对不能嵌套； 即，FU 一定不能包含另一个 FU 的子集。

携带 FU 的 RTP 分组的 RTP 时间戳被设置为分段 NAL 单元的 NALU 时间。

FU 由一个有效负载报头（PayloadHdr），一个 8bit 的 FU Header，一个有条件的 16 位 DONL 字段和 FU 有效负载组成，如下图所示。

    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |    PayloadHdr (Type=49)       |   FU header   | DONL (cond)   |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-|
    | DONL (cond)   |                                               |
    |-+-+-+-+-+-+-+-+                                               |
    |                         FU payload                            |
    |                                                               |
    |                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                               :...OPTIONAL RTP padding        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                Figure 9: The Structure of an FU

PayloadHdr 的字段含义与 NALU Header 的字段含义一直，其各字段设置为：
    - Type 字段必须等于 49
    - 字段 F，LayerId 和 TID 必须分别等于分段的 NAL 单元的字段 F，LayerId 和 TID

FU header 包含 1bit 的字段 S，1 字节的字段 E 和 6bit 的字段 FuType：

            +---------------+
            |0|1|2|3|4|5|6|7|
            +-+-+-+-+-+-+-+-+
            |S|E|  FuType   |
            +---------------+

    Figure 10: The Structure of FU Header

- S: 1bit 当设置为 1 时，S 位指示分段 NAL 单元的开始，即 FU 有效载荷的第一个字节也是分段 NAL 单元的有效载荷的第一个字节。 当 FU 有效载荷不是分段式 NAL 单元有效载荷的开始时，必须将 S 位设置为 0。
- E: 1bit 当设置为 1 时，E 位表示分段 NAL 单元的末尾，即有效载荷的最后一个字节也是分段 NAL 单元的最后一个字节。 当 FU 有效负载不是分段的 NAL 单元的最后一个分段时，E 位务必设置为 0。
- FuType: 6bit 字段 FuType 必须等于分段的 NAL 单元的字段 Type。

## 参考资料

* [1] [RFC 7798 : RTP Payload Format for High Efficiency Video Coding (H265)](https://www.rfc-editor.org/rfc/rfc7798.txt)
