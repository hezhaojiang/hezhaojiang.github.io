---
title: 音视频封装 - PS 封装格式
categories:
  - 音视频封装
tags:
  - MPEG2-PS
abbrlink: 9064839e
date: 2020-07-28 16:32:42
---
## MPEG2-PS 简介

PS 流（MPEG program stream）是用于多路复用数字音频、视频等的容器格式。PS 格式在 MPEG-1 Part 1 (ISO/IEC 11172-1) 和 MPEG-2 Part 1, Systems (ISO/IEC standard 13818-1/ITU-T H.222.0) 中指定。

<!-- more -->

## 格式

### Program Stream

![Program Stream 结构](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200729221633.jpg)

> pack_start_code – The pack_start_code is the bit string '0000 0000 0000 0000 0000 0001 1011 1010' (0x000001BA). It identifies the beginning of a pack.
> MPEG_program_end_code – The MPEG_program_end_code is the bit string '0000 0000 0000 0000 0000 0001 1011 1001' (0x000001B9). It terminates the Program Stream.

### Program Stream Pack

![Program Stream Pack 结构](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200729222439.png)

> packet_start_code_prefix – The packet_start_code_prefix is a 24-bit code. Together with the stream_id that follows it constitutes a packet start code that identifies the beginning of a packet. The packet_start_code_prefix is the bit string '0000 0000 0000 0000 0000 0001' (0x000001).

### Program Stream Pack Header

![Program Stream Pack Header 结构](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200729222901.jpg)

* pack_start_code — pack_start_code 为比特串'0000 0000 0000 0000 0000 0001 1011 1010' (0x000001BA)。它标识 PS 包的起始。
* system_clock_reference_base; system_clock_reference_extension — 系统时钟参考（SCR）为分成两部分编码的 42 比特字段。第一部分 system_clock_reference_base 为 33 比特字段，第二部分 system_clock_reference_extension 为 9 比特字段。SCR 指示在节目目标解码器的输入端包含 system_clock_reference_base 最后比特的字节到达的预期时间。
* marker_bit — marker_bit 为 1 比特字段，其值为‘1’。
* program_mux_rate — 此为 22 比特整数，指示包期间 P-STD 接收节目流的速率，其中该节目流包含在包中。program_mux_rate 值以 50 字节 / 秒为度量单位。0 值禁用。program_mux_rate 中表示的值用于规定字节到达 2.5.2 中的 P-STD 输入端的时间。在 program_mux_rate 字段中的编码值可以随着 ITU-T H.222.0 建议书 | ISO/IEC 13818-1 节目多路复用流中的包到包的变化而改变。
* pack_stuffing_length — 3 比特整数，指示跟随此字段的填充字节数。
* stuffing_byte — 此为等于‘1111 1111’的固定 8 比特 值，可以由编码器插入，例如满足信道的要求。它由解码器丢弃。在每个包头中，应存在不多于 7 个的填充字节。

### System header

![System header 结构](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200729224213.png)

* system_header_start_code — 比特串 '0000 0000 0000 0000 0000 0001 1011 1011' (0x000001BB)。它标识 System header 的起始。
* header_length — 此 16 比特字段指示跟随 header_length 字段的系统头的字节长度。
* rate_bound — 22 比特字段。rate_bound 为大于或等于在任意节目流包中编码的 program_mux_rate 字段的最大值的整数值。它可供解码器使用来评估它是否有能力解码该完整流。
* audio_bound — 6 比特字段。为 0 到 32 闭区间内的一个整数，在解码过程同时被激活的 PS 流中，它被设置为大于或等于 ISO/IEC 13818-3 和 ISO/IEC 11172-3 音频流最大数的整数值。
* fixed_flag — fixed_flag 为 1 比特标志。置于‘1’时指示固定的比特速率操作。置于‘0’时指示可变比特速率操作。
* CSPS_flag — CSPS_flag 为 1 比特字段。若其值置于‘1’，则节目流满足 2.7.9 中规定的限制。
* system_audio_lock_flag — system_audio_lock_flag 为 1 比特字段，指示音频采样速率和系统目标解码器的 system_clock_frequency 之间存在特定的常量比率关系。
* system_video_lock_flag — system_video_lock_flag 为 1 比特字段，指示视频时间基和系统目标解码器的系统时钟频率之间存在特定的常量比率关系。
* video_bound — video_bound 为 5 比特整数，在 0 到 16 的闭区间内取值，在解码过程同时被激活的 PS 流中，它被设置为大于或等于视流的最大数的整数值。
* packet_rate_restriction_flag — 1 比特标志。
* reserved_bits — 此 7 比特字段由 ISO/IEC 保留供未来使用。值应为‘111 1111’。
* stream_id — stream_id 为 8 比特字段， 指示以下 P-STD_buffer_bound_scale 和 P-STD
* P-STD_buffer_size_bound 字段所涉及的流的编码与基本流编号。若 stream_id 等于‘ 1011 1000 ’， 则跟随 stream_id 的 P-STD_buffer_bound_scale 和 P-STD_buffer_size_bound 字段涉及节目流中的所有音频流。若 stream_id 等于‘ 1011 1001 ’， 则跟随 stream_id 的 P-STD_buffer_bound_scale 和 P-STD_buffer_size_bound 字段涉及节目流中的所有视频流。
若 stream_id 取任何其他值，则它将是大于或等于‘1011 1100’的字节值并将解释为涉及依照表 2-22
的流编码和基本流编号。
* P-STD_buffer_bound_scale — P-STD_buffer_bound_scale 为 1 比特字段， 指示用于解释后续
* P-STD_buffer_size_bound 字段的标度因子。若前导 stream_id 指示音频流，则 P-STD_buffer_bound_scale 必有 0 值。若前导 stream_id 指示视频流，则 P-STD_buffer_bound_scale 必有 1 值。对所有其他流类型，P-STD_buffer_bound_scale 的值可以为 1 或为 0。
* P-STD_buffer_size_bound — P-STD_buffer_size_bound 为 13 比特无符号整数，规定该值大于或等于 PS 流中流 n 的所有包上的最大 P-STD 输入缓冲器尺寸 BSn。若 P-STD_buffer_bound_scale 有 0 值，那
么 P-STD_buffer_size_bound 以 128 字节为单位度量该缓冲器尺寸限制。若 P-STD_buffer_bound_scale 有‘1’
值，那么 P-STD_buffer_size_bound 以 1024 字节为单位度量该缓冲器尺寸限制。

## 总结

PS 码流格式总结如下图：

![PS码流格式](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201214231955.jpg)

> PES Packet 参见 {% post_link "音视频封装 - PES 封装格式" %}

## 参考资料

* [1] MPEG-2 Part 1, Systems
