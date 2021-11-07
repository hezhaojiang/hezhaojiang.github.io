---
title: 编解码技术：H264 - NAL Unit
categories:
  - 编解码技术
tags:
  - H.264
abbrlink: f2a4ae37
date: 2020-08-10 14:19:28
---
## NAL Unit 简介

NAL(Network Abstract Layer), 即网络抽象层。

在 H.264/AVC 视频编码标准中，整个系统框架被分为了两个层面：视频编码层面（VCL, Video Coding Layer）和网络抽象层面（NAL, Network Abstract Layer）。其中，前者负责有效表示视频数据的内容，而后者则负责格式化数据并提供头信息，以保证数据适合各种信道和存储介质上的传输。

NAL unit 是 NAL 的基本语法结构，它包含一个字节的头信息和一系列来自 VCL 的称为拓展字节序列载荷（EBSP）的字节流。

<!-- more -->

## NAL Unit 结构

H.264 标准中规定的一个 NAL Unit 的结构如下图：

![NAL Unit 结构](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200810224434.png)

从上图可看出，NAL Unit 的结构主要分为两部分：

1. NAL Header
2. NAL Body

### NAL Header

对于基本版本的 H.264 标准（不考虑 SVC 和 MVC 扩展），一个 NAL Header 的长度固定为 1 字节，即 8bit。

![NAL Header](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200810225427.png)

这 8bit 的含义分别如下：

- forbidden_zero_bit：每一个 NAL Header 的第一个 bit，规定必须为 0；
- nal_ref_idc：第 2 和 3 位，主要表示 NAL 的优先级。当该值不等于 0 时，表示当前 NAL Unit 中包含了 SPS、PPS 和作为参考帧的 Slice 等重要数据。

    - 对于 nal_unit_type 等于 7、8、13、15 的 NAL 单元，nal_ref_idc 不得等于 0
    - 对于 nal_unit_type 等于 5 的 NAL 单元，nal_ref_idc 不得等于 0。
    - 对于所有 nal_unit_type 等于 6、9、10、11 或 12 的 NAL 单元，nal_ref_idc 均应等于 0。

- nal_unit_type：表示 NAL Unit 的类型，包括 VCL 层和非 VCL 层的多种数据类型。常见的 nal_unit_type 取值见下表：

<!--
![常见 nal_unit_type](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200810225932.png)
-->

![常见 nal_unit_type](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200811221628.png)

### NAL Body

在 NAL Header 之后，NAL Unit 的其余部分，即 NAL Body 包含了有效负载数据的封装。从 NAL Body 到实际的语法元素的码流共 3 层封装：

#### EBSP（扩展字节序列载荷）

EBSP 全称为 Extended Byte String Payload，等同于 NAL Body 的数据本身。在 EBSP 中包含了一个特殊的字节 emulation_prevention_three_byte(0x03)，表示防止竞争校验字节：

emulation_prevention_three_byte：0x03，设置该值的目的是为了防止 NAL Body 内部出现于 NAL Unit 起始码 0x 00 00 01 或 0x 00 00 00 01 冲突。

当内部的连续 3 字节数据出现了下列情况时：

    · 0x 00 00 00
    · 0x 00 00 01
    · 0x 00 00 02
    · 0x 00 00 03

在两个 0 字节之后会插入值为 3 的一个字节，形成下列情况：

    · 0x 00 00 03 00
    · 0x 00 00 03 01
    · 0x 00 00 03 02
    · 0x 00 00 03 03

在进行解析时需要将附加的 03 字节去掉，得到 RBSP 数据。

#### RBSP（原始字节序列载荷）

RBSP 全称为 Raw Byte Sequence Payload，相当于 NAL Body 去掉 emulation_prevention_three_byte 之后的数据，是对原始的语法元素码流进一步处理后产生的数据。相比于原始的语法元素码流，RBSP 在末尾添加了 rbsp_trailing_bits() 部分，其主要目的是字节对齐。

rbsp_trailing_bits 结构如下图所示：

![rbsp_trailing_bits](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200810232046.png)

从图中可以看出，每个 rbsp_trailing_bits() 包括一个 1bit 和若干个 0bit，0bit 的个数不定，以实现字节的对齐。

#### SODB（数据比特串）

SODB 全称为 String Of Data Bits，表示 H.264 的语法元素编码完成后的实际的原始二进制码流。SODB 通常不能保证字节对齐。

## NAL Unit 的传输

在 H264 编码时，为了方便解码器识别 NAL Unit 的位置，在每个 NAL Unit 前添加3个字节的起始码 0x000001，解码器一旦在码流中检测到起始码，这就说明刚刚解码的 NAL Unit 结束，新的 NAL Unit 即将开始。

为防止 NAL Unit 内部出现 0x000001 数据，H264 采取了一种防竞争机制，即使用 EBSP 替代 RBSP（见上文）

## 参考资料

* [1] 《 T-REC-H.264-201906-I.pdf 》
* [2] 《 新一代视频压缩编码标准 -- H264/AVC（第二版）》
* [3] 《 H.265/HEVC 视频编码新标准及其拓展 》
