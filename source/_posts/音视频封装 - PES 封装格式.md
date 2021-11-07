---
title: 音视频封装 - PES 封装格式
categories:
  - 音视频封装
tag:
  MPEG2-PES
abbrlink: f33d2f13
date: 2020-07-29 15:29:34
---
## 简介

PES(Packetized elementary stream) 是 MPEG-2 Part 1, Systems（ISO/IEC 13818-1） 和 ITU-T H.222.0 中的规范，它们定义了 PS(MPEG program streams) 和 TS(MPEG transport streams) 中数据包中 ES(elementary stream)（通常是音频或视频编码器的输出）的携带。 通过将 PES 数据包头内 ES 的顺序数据字节封装组成 PES。

<!-- more -->

## 格式

### PES packet

![PES Packet](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200729232326.jpg)

* **Packet start code prefix**：包头起始码，固定为 0x000001，占位 24bit；

* **Stream id**：PES 包中的负载流类型，一般视频为 0xe0，音频为 0xc0，占位 8bit；

* **PES packet length**：PES 包长度，包括此字节后的可选包头和负载的长度，占位 16bit；

* Optional PES Header，顺序依次为：

    - "10" 字段：占位 2bit；
    - PES scrambling control：加密模式，占 2bit；00 未加密，01 或 10 或 11 由用户定义；
    - PES priority：有效负载的优先级，占位 1bit；值为 1 比值为 0 的负载优先级高；
    - Data alignment indicator：数据定位指示器，占位 1bit；
    - Copyright：版权信息，1 为有版权，0 无版权，占位 1bit；
    - Original or copy：原始或备份，1 为原始，0 为备份，占位 1bit；
    - *后面是 7 个 flags(一般我们关注的就是 PTS DTS 的标志位)：*
    - PTS_DTS_flags：PTS 和 DTS 标志位，占位 2bit；10 表示首部有 PTS 字段，11 表示有 PTS 和 DTS 字段，00 表示都没有，01 被禁止，不会出现此种情况；
    - ESCR_flag：ESCR 标志，占位 1bit；1 表示首部有 ESCR 字段，0 则无此字段
    - ES_rate_flag：ES_rate 字段，占位 1bit；1 表示首部有此字段，0 无此字段；
    - DSM_trick_mode_flag：占位 1bit；1 表示有 8 位的 DSM_trick_mode_flag 字段，0 无此字段；
    - Additional_copy_info_flag：占位 1bit；1 表示首部有此字段，0 表示无此字段；
    - PES_CRC_flag：占位 1bit；置 1 表示 PES 分组有 CRC 字段，0 无此字段；
    - PES_extension_flag：占位 1bit；扩展标志位，置 1 表示有扩展字段，0 无此字段；
    - PES header data length：(UI)PES 首部中可选字段和填充字段的长度；占位 8bit；可选字段的内容由上面 7 个 flags 来进行控制；

* Optional fields：可选字段的描述信息区域，其内容由上面的 7 个 flag 来控制；

    - **PTS/DTS 字段**：显示时间戳 / 解码时间戳，占位 (1~2) * 40bit，当 `PTS_DTS_flags == 11` 时此字段存在 2 个，占 80bit，当 `PTS_DTS_flags == 10` 时此字段存在 1 个，占 40bit；时间占用 33 个 bit，PTS 和 DTS 的内容是在这 40bit 中取 33 位，方式相同；
        * *PTS(presentation time stamp) 显示时间戳和 DTS(Decoding Time Stamp) 解码时间戳，是用来音视频同步的，是打在 PES 包的包头里面的，PTS/DTS 是相对 SCR(系统参考) 的时间戳，是以 90000 为单位的，PTS/DTS 到 ms 的转换公式是 PTS/90，系统时钟频率 (H264 采样频率?) 为 90Khz，所以转换到秒为 PTS/90000，所以如果是以 ms 为单位的播放器，PTS/DTS 是要使用公式 ms=pts/90 来转换才行的，而如果是以时钟频率为单位的话，则直接将 PTS/DTS 送进去解码即可；如果没有 B 帧，PTS 和 DTS 的顺序应该是一致的，如果有 B 帧，则需要先解码 P 帧，才能解出来 B 帧，所以需要 PTS 和 DTS 来控制解码时间和显示时间；*
        * start_code：起始码，占位 4bit；若 `PTS_DTS_flags == '10'` ，则说明只有 PTS；若 `PTS_DTS_flags == '11'` ，则 PTS 和 DTS 都存在，PTS 的起始码为 0011，DTS 的起始码为 0001；
        * PTS[32..30]：占位 3bit；
        * marker_bit：占位 1bit；
        * PTS[29..15]：占位 15bit；
        * marker_bit：占位 1bit；
        * PTS[14..0]：占位 15bit；
        * marker_bit：占位 1bit；
        * *PTS/DTS  = （PTS1 & 0x0e) <<29 + (PTS2 & 0xfffe) << 14 + (PTS3 & 0xfffe ) >> 1;*

    - ESCR 字段：此字段占位 48bit，由 33bit 的 ESCR_base 字段和 9bit 的 ESCR_extension 字段组成， `ESCR_flag == 1` 时此字段存在；数据依次顺序：
        * Reserved：保留字段，占位 2bit；
        * ESCR_base[32..30]：占位 3bit；
        * marker_bit：占位 1bit；
        * ESCR_base[29..15]：占位 15bit；
        * marker_bit：占位 1bit；
        * ESCR_base[14..0]：占位 15bit；
        * marker_bit：占位 1bit；
        * ESCR_extension：占位 9bit；周期数，取值范围 0~299；循环一次，base+1；
        * marker_bit：占位 1bit；

    - ES rate 字段：目标解码器接收 PES 分组字节速率，禁止为 0，占位 24bit， `ES_rate_flag == 1` 时此字段存在；数据顺序为：
        * marker_bit：占位 1bit；
        * ES_rate：占位 22bit；
        * marker_bit：占位 1bit；

    - Trick mode control 字段：表示哪种 trick mode 被应用于相应的视频流，占位 8bit， `DSM_trick_mode_flag == 1` 时此字段存在；其中 trick_mode_control 占前 3bit，根据其值后面有 5bit 的不同内容；
        * 如果 trick_mode_control == ‘000’，依次字节顺序为：
            - field_id：占位 2bit；
            - intra_slice_refresh ：占 1bit；
            - frequency_truncation：占 2bit；
        * 如果 trick_mode_control == ‘001’，依次字节顺序为：
            - rep_cntrl：占位 5bit；
        * 如果 trick_mode_control == ‘010’，依次字节顺序为：
            - field_id：占位 2bit；
            - Reserved：占位 3bit；
        * 如果 trick_mode_control == ‘011’，依次字节顺序为：
            - field_id：占位 2bit；
            - intra_slice_refresh：占 1bit；
            - frequency_truncation：占 2bit；
        * 如果 trick_mode_control== ‘100’，依次字节顺序为：
            - rep_cntrl：占位 5bit；
        * 其他情况，字节顺序为：
            - reserved ：占位 5bit；

    - Additional copy info 字段：占 8 个 bit， `Additional_copy_info_flag == 1` 时此字段存在；数据顺序为：
        * marker_bit：占位 1bit；
        * copy info 字段：占位 7bit；表示和版权相关的私有数据；

    - Previous PES CRC 字段：占位 16bit 字段，包含 CRC 值， `PES_CRC_flag == 1` 时此字段存在；

    - PES extension 字段：PES 扩展字段， `PES_extension_flag == 1` 时此字段存在；内容如下，字节顺序依次为：
        * PES_private_data_flag：占 1bit，置 1 表示有私有数据，0 则无；
        * Pack_header_field_flag：占 1bit，置 1 表示有 Pack_header_field 字段，0 则无；
        * Program_packet_sequence_counter_flag：占 1bit，置 1 表示有此字段，0 则无；
        * P-STD_buffer_flag：占位 1bit，置 1 表示有 P-STD_buffer 字段，0 则无此字段；
        * Reserved 字段：3bit；
        * PES_extension_flag_2：占位 1bit，置 1 表示有扩展字段，0 则无此字段；

    - Optional field ：PES 扩展字段的可选字段内容顺序为：
        * PES_private_data 字段：私有数据内容，占位 128bit， `PES_private_data_flag == 1` 时此字段存在；
        * Pack_header_field 字段： `Pack_header_field_flag == 1` 时此字段存在；字段组成顺序如下：
            - Pack_field_length 字段：(UI) 指定后面的 field 的长度，占位 8bit；
            - pack_header_field()：长度为 Pack_field_length 指定；
        * Program_packet_sequence_counter 字段：计数器字段，16 个 bit；当 flag 字段 `Program_packet_sequence_counter_flag == 1` 时此字段存在；字节顺序依次为：
            - marker_bit：占位 1bit；
            - packet_sequence_counter 字段：(UI) 占位 7bit；
            - marker_bit：占位 1bit；
            - MPEG1_MPEG2_identifier：占位 1bit；置位 1 表示此 PES 包的负载来自 MPEG1 流，置位 0 表示此 PES 包的负载来自 PS 流；
            - original_stuff_length：(UI) 占位 6bit；表示 PES 头部填充字节长度；
        * P-STD_buffer 字段：表示 P-STD_buffer 内容，占位 16bit； `P-STD_buffer_flag == '1'` 时此字段存在；字节顺序依次为：
            - '01'字段：占位 2bit；
            - P-STD_buffer_scale：占位 1bit；表示用来解释后面 P-STD_buffer_size 字段的比例因子；如果之前的 stream_id 表示音频流，则此值应为 0，若之前的 stream_id 表示视频流，则此值应为 1，对于其他 stream 类型，此值可以 0 或 1；
            - P-STD_buffer_size：占位 13bit；无符号整数；大于或等于所有 P-STD 输入缓冲区大小 BSn 的最大值；若 P-STD_buffer_scale == 0，则 P-STD_buffer_size 以 128 字节为单位；若 P-STD_buffer_scale == 1，则 P-STD_buffer_size 以 1024 字节为单位；
        * PES_extension2 字段：扩展字段的扩展字段；占用 N*8 个 bit， `PES_extension_flag_2 == '1'` 时此字段存在；字节顺序依次为：
            - marker_bit：占位 1bit；
            - PES_extension_field_length：占位 7bit，表示扩展区域的长度；
            - Reserved 字段：占位 8*PES_extension_field_length 个 bit；

    - Stuffing bytes：填充字段，固定为 0xFF；不能超过 32 个字节；

    - PES_packet_data_byte：PES 包负载中的数据，即 ES 原始流数据；

## 参考资料

* [1] MPEG-2 Part 1, Systems
* [2] [PES,TS,PS,RTP等流的打包格式解析之PES流_appledurian的博客](https://blog.csdn.net/appledurian/article/details/70851428)
* [3] [Packetized elementary stream - Wikipedia](https://en.wikipedia.org/wiki/Packetized_elementary_stream)
