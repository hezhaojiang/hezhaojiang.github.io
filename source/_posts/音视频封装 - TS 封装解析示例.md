---
title: 音视频封装 - TS 封装解析示例
categories:
  - 音视频封装
tags:
  - MPEG2-TS
abbrlink: a03a01c2
date: 2020-11-27 16:13:10
---
文章 {% post_link "音视频封装 - TS 封装格式" %} 中分析了 `TS` 流的结构，本文通过一段真实的码流的分析来加强对 `TS` 流结构的理解。

> 注 1：本文中的 `TS` 码流截取自一段从网络下载的网络视频

<!-- more -->

## PAT 包解析

该段码流中第一包并不是 `PAT` 包，而 TS 封装的解析需要从 `PAT` 包开始，所以我们向后寻找 PID 为 0 的第一包，该段码流中第二包 PID 为 0，是一个 `PAT` 包：

![Packet 2](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201128145341.png)

第一部分：`TS` 包头

    HEX 47       40       00       10
    BIN 01000111 01000000 00000000 00010000

对比该部分结构，可代码定义如下：

``` cpp
typedef struct TS_HEAD {
    unsigned sync_byte                    : 8;  // 同步字节，固定为 0x47
    unsigned transport_error_indicator    : 1;  // 传输错误指示符，0 表明在 TS 头的 adapt 域后没有无用字节
    unsigned payload_unit_start_indicator : 1;  // 负载单元起始标示符，一个完整的数据包开始时标记为 1
    unsigned transport_priority           : 1;  // 传输优先级，0 为低优先级
    unsigned pid                          : 13; // PID 值，0 表示 TS 头后面就是 PAT 表
    unsigned transport_scrambling_controls    : 2;  // TS 流 ID，一般为 0x0001，区别于一个网络中其它多路复用的流
    unsigned adaptation_field_control         : 2;  // 是否包含自适应区，'01'为无自适应域，仅含有效负载
    unsigned continuity_counter               : 4;  // 递增计数器，0-F，起始值不一定取 0，但必须是连续的
} TS_HEAD;
```

第二部分：`TS` 包调整字节

    HEX 00
    BIN 00000000

对比该部分结构，代码定义分析如下：

``` cpp
typedef struct TS_ADAPT {
    unsigned char adaptation_field_length;  // 自适应域长度，后面的字节数，此包中为 7 字节
    unsigned char flag;                     // 0x50 表示包含 PCR，0x40 表示不包含 PCR
    if (flag == 0x50) unsigned char PCR[5]; // 节目时钟参考 40bit
    vector<unsigned char> stuffing_bytes;   // 填充字节，取值 0xFF
}
```

`adaptation_field_length` 为 `0`，故该部分后续字节数为 `0`。

第三部分：`TS` 包有效载荷 - 即 `PSI` 表 `PAT` 的数据：

    HEX 00 B0 0D 00 01 C1 00 00  00 01 F0 00 2A B1 04 B2
    BIN 00000000 10110000 00001101 00000000 00000001 11000001 00000000 00000000
        00000000 00000001 11110000 00000000 00101010 10110001 00000100 10110010

对比该部分结构，代码定义分析如下：

``` cpp
typedef struct TS_PAT {
    unsigned table_id                     : 8;  // 固定 0x00 ，标志是该表是 PAT
    unsigned section_syntax_indicator     : 1;  // 段语法标志位，固定为 1
    unsigned zero                         : 1;  // 0
    unsigned reserved_1                   : 2;  // 保留位，固定为 11
    unsigned section_length               : 12; // 段长度，0000 00001101 值为 13
    unsigned transport_stream_id          : 16; // TS 流 ID，一般为 0x0001，区别于一个网络中其它多路复用的流
    unsigned reserved_2                   : 2;  // 保留位，固定为 11
    unsigned version_number               : 5;  // PAT 版本号，固定为 00000，如果 PAT 有变化则版本号加 1
    unsigned current_next_indicator       : 1;  // 固定为 1，表示这个 PAT 表有效，如果为 0 则要等待下一个 PAT 表
    unsigned section_number               : 8;  // 分段的号码。PAT 可能分为多段传输，第一段为 00，以后每个分段加 1，最多可能有 256 个分段
    unsigned last_section_number          : 8;  // 最后一个分段的号码
    std::vector<TS_PAT_Program> program; = {    // 该包中仅有一个 TS_PAT_Program
        {
            unsigned program_number       :16;  // 节目号，为 0x0000 时表示这是 NIT，节目号为 0x0001 时, 表示这是 PMT
            unsigned reserved             :3    // 保留，固定为 111
            if (program_number == 0x0000)
                unsigned network_PID      :13;
            else if (program_number == 0x0001)
                unsigned program_map_PID  :13;  // 节目号对应内容的 PID 值，该 PAT 包取值为 0x1000 = 4096
            }
        },
    };
    unsigned CRC_32                       : 32; // CRC32 校验码
} TS_PAT;
```

## PMT 包解析

`PAT` 信息中可知 `PMT` 表的 `PID` 为 `4096`，我们向后寻找 `PID` 为 `4096` 的 `TS` 包，发现第 `3` 包 `TS` 包即为 `PMT` 包：

![PMT](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201128155219.png)

第一部分：`TS` 包头

    HEX 47       50       00       10
    BIN 01000111 01010000 00000000 00010000

从 `TS` 包头可知该 `TS` 包的 `PID` 值为 `4096`，是我们寻找的 `PMT` 包。

第二部分：`TS` 包调整字节，该部分只有一个字节 `0x00`，详见上文解析，此处不再重复。

第三部分：`TS` 包有效载荷 - `PMT`

    HEX 02 B0 23 00 01 C1 00 00  E1 00 F0 00 24 E1 00 F0
        06 05 04 48 45 56 43 03  E1 01 F0 06 0A 04 75 6E
        64 00 DD 33 FC 6A
    BIN 00000010 10110000 00100011 00000000 00000001 11000001 00000000 00000000
        11100001 00000000 11110000 00000000 00100100 11100001 00000000 11110000
        00000110 00000101 00000100 01001000 01000101 01010110 01000011 00000011
        11100001 00000001 11110000 00000110 00001010 00000100 01110101 01101110
        01100100 00000000 11011101 00110011 11111100 01101010

对比该部分结构，代码定义分析如下：

``` cpp
typedef struct TS_PMT {
    unsigned table_id                     : 8;  // 取值随意，一般使用 0x02, 表示 PMT 表
    unsigned section_syntax_indicator     : 1;  // 段语法标志位，固定为 1
    unsigned zero                         : 1;  // 保留位，固定为 0
    unsigned reserved_1                   : 2;  // 保留位，固定为 11
    unsigned section_length               : 12; // 段长度，0000 00100011 = 35 字节
    unsigned program_number               : 16; // 当前 PMT 表映射到的节目号，1、2、3
    unsigned reserved_2                   : 2;  // 保留位，固定为 11
    unsigned version_number               : 5;  // PMT 版本号码，固定为 00000，如果 PAT 有变化则版本号加 1
    unsigned current_next_indicator       : 1;  // 发送的 PMT 表， 1 当前有效 PMT 有效
    unsigned section_number               : 8;  // 分段的号码。PMT 可能分为多段传输，第一段为 00，以后每个分段加 1，最多可能有 256 个分段
    unsigned last_section_number          : 8;  // 分段数
    unsigned reserved_3                   : 3;  // 保留位，固定为 111
    unsigned PCR_PID                      : 13; // 指明 TS 包的 PID 值 (0x0100 = 256)，该 TS 包含有 PCR 同步时钟，
    unsigned reserved_4                   : 4;  // 预留位，固定为 1111
    unsigned program_info_length          : 12; // 前 2bit 为 00，该域指出跟随其后对节目信息的描述的字节数 (0x000 = 0)。
    std::vector<unsigned char> descriptor;      // 0 字节
    std::vector<TS_PMT_Stream> PMT_Stream = {
        {
            unsigned stream_type          : 8;  // 指示本节目流的类型，H.265 编码对应 0x24
            unsigned reserved1            : 3;  // 保留位，固定为 111
            unsigned elementary_PID       : 13; // 指示该流的 PID 值 (0x100 = 256)
            unsigned reserved2            : 3;  // 保留位，固定为 1111
            unsigned ES_info_length       : 12; // 前两位 bit 为 00，指示跟随其后的描述相关节目元素的字节数 (0x006 = 6)
            std::vector<unsigned char> descriptor;   // 6 字节
        },
        {
            unsigned stream_type          : 8;  // 指示本节目流的类型，MP3 编码对应 0x03
            unsigned reserved1            : 3;  // 保留位，固定为 111
            unsigned elementary_PID       : 13; // 指示该流的 PID 值 (0x101 = 257)
            unsigned reserved2            : 3;  // 保留位，固定为 1111
            unsigned ES_info_length       : 12; // 前两位 bit 为 00，指示跟随其后的描述相关节目元素的字节数 (0x006 = 6)
            std::vector<unsigned char> descriptor;   // 6 字节
        },
    };
    unsigned CRC_32                       : 32; // CRC32 校验码
} TS_PMT;
```

`PMT` 是定义每路节目的音视频类型 `TYPE` 和编号 `PID` 的关键，该 `TS` 码流中：
    - 视频流格式为 `H.265`，对应 `PID` 为 `256`
    - 视频流格式为 `MP3`，对应 `PID` 为 `257`

## 视频包解析

第 `4` 包的 `PID` 值为 `256`，根据上述分析可知，该包为一个视频包：

![视频包](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201128163234.png)

第一部分：`TS` 包头

    HEX 47       41       00       30
    BIN 01000111 01000001 00000000 00110000

从 `TS` 包头可知该 `TS` 包的 `PID` 值为 `256`，是我们寻找的视频包，另外，TS 包头中 `adaptation_field_control` 字段为 `11`，代表同时带有自适应域和有效负载。

第二部分：`TS` 包调整字节

    HEX 07 50 00 00 7C F7 7E 00
    BIN 00000111 01010000 00000000 00000000 01111100 11110111 01111110 00000000

对比该部分结构，代码定义分析如下：

``` cpp
typedef struct TS_ADAPT {
    unsigned char adaptation_field_length;  // 自适应域长度，后面的字节数，此包中为 7 字节
    unsigned char flag;                     // 0x50 表示包含 PCR
    if (flag == 0x50) unsigned char PCR[5]; // 节目时钟参考 40bit
    vector<unsigned char> stuffing_bytes;   // 填充字节，取值 0xFF
}
```

第三部分：`TS` 包有效载荷 - `PES`

    HEX 00 00 01 E0 00 00 80 80  05 21 00 07 E0 0D
    BIN 00000000 00000000 00000001 11100000 00000000 00000000 10000000 10000000
        00000101 00100001 00000000 00000111 11100000 00001101

对比该部分结构，代码定义分析如下：

``` cpp
typedef struct TS_ADAPT {
    unsigned start_code_prefix        : 24; // 包头起始码，固定为 0x000001
    unsigned stream_id;               : 8;  // PES 包中的负载流类型，一般视频为 0xe0，音频为 0xc0
    unsigned PES_packet_length        : 16; // PES 包长度，包括此字节后的可选包头和负载的长度
    unsigned reserved_1               : 2;  // 保留位，固定为 10
    unsigned PES_scrambling_control   : 2;  // 加密模式，00 未加密，01 或 10 或 11 由用户定义
    unsigned PES_priority             : 1;  // 有效负载的优先级，值为 1 比值为 0 的负载优先级高
    unsigned Data_alignment_indicator : 1;  // 数据定位指示器
    unsigned Copyright                : 1;  // 版权信息，1 为有版权，0 无版权
    unsigned Original_or_copy         : 1;  // 原始或备份，1 为原始，0 为备份
    unsigned PTS_DTS_flags            : 2;  // PTS 和 DTS 标志位，10 表示首部有 PTS 字段，11 表示有 PTS 和 DTS 字段，00 表示都没有，01 被禁止
    unsigned ESCR_flag                : 1;  // ESCR 标志，1 表示首部有 ESCR 字段，0 则无此字段
    unsigned ES_rate_flag             : 1;  // ES_rate 字段，1 表示首部有此字段，0 无此字段
    unsigned DSM_trick_mode_flag      : 1;  // 1 表示有 8 位的 DSM_trick_mode_flag 字段，0 无此字段
    unsigned Additional_copy_info_flag: 1;  // 1 表示首部有此字段，0 表示无此字段
    unsigned PES_CRC_flag             : 1;  // 1 表示 PES 分组有 CRC 字段，0 无此字段
    unsigned PES_extension_flag       : 1;  // 扩展标志位，置 1 表示有扩展字段，0 无此字段
    unsigned PES_header_data_length   : 8;  // PES 首部中可选字段和填充字段的长度，可选字段的内容由上面 7 个 flags 来进行控制
    if (PTS_DTS_flags == '10') {
        unsigned reserved_1           : 4;  // 保留位，固定为 0010
        unsigned PTS_32_30            : 3;  // PTS
        unsigned marker_1             : 1;  // 保留位，固定为 1
        unsigned PTS_29_15            : 15; // PTS
        unsigned marker_2             : 1;  // 保留位，固定为 1
        unsigned PTS_14_0             : 15; // PTS
        unsigned marker_3             : 1;  // 保留位，固定为 1
    }
    if (PTS_DTS_flags == '11') {
        unsigned reserved_1           : 4;  // 保留位，固定为 0011
        unsigned PTS_32_30            : 3;  // PTS
        unsigned marker_1             : 1;  // 保留位，固定为 1
        unsigned PTS_29_15            : 15; // PTS
        unsigned marker_2             : 1;  // 保留位，固定为 1
        unsigned PTS_14_0             : 15; // PTS
        unsigned marker_3             : 1;  // 保留位，固定为 1
        unsigned reserved_2           : 4;  // 保留位，固定为 0001
        unsigned DTS_32_30            : 3;  // DTS
        unsigned marker_1             : 1;  // 保留位，固定为 1
        unsigned DTS_29_15            : 15; // DTS
        unsigned marker_2             : 1;  // 保留位，固定为 1
        unsigned DTS_14_0             : 15; // DTS
        unsigned marker_3             : 1;  // 保留位，固定为 1
    }
    if (ESCR_flag) ...
    if (ES_rate_flag) ...
    ...
    if (PES_extension_flag) ...
    vector<unsigned char> stuffing_bytes;   // 填充字段，固定为 0xFF，不得超过 32 字节
    vector<unsigned char> PES_packet_data_byte // PES 包的负载数据
}
```

对比代码和码流数据，发现码流中 `PES_packet_length` 值为 `0`，其表示 `PES` 分组长度要么没有规定要么没有限制。这种情况只允许出现在有效负载包含来源于传输流分组中某个视频基本流的字节的 `PES` 分组中。

该段视频流中 `PTS_DTS_flags == '10'`，故其还有 `5` 字节的 `PTS` 信息，我们尝试解析 `PTS` 信息：

1. `PTS` 的 `5` 字节信息为 `21 00 07 E0 0D`，对应二进制 `00100001 00000000 00000111 11100000 00001101`
2. 除去 `reserved` 和 `marker` 字段为 `000 00000000 0000011 11100000 0000110`
3. 整理可得 `0 00000000 00000001 11110000 00000110`，即 `0x001F006`
4. 转为十进制为 `126982`，与工具计算结果一致

`PES` 包头数据分析完，剩下的数据就是帧视频数据的一部分了。

## 音频包解析

音频包与视频包均使用 `PES` 格式封装，解析方式基本一致，比如：

![音频包](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201128192950.png)

音频包中第三部分：`TS` 包有效载荷 - `PES`

    HEX 00 00 01 C0 09 D3 80 80 05 21 00 07 D8 61 FF
    BIN 00000000 00000000 00000001 11000000 00001001 11010011 10000000 10000000
        00000101 00100001 00000000 00000111 11011000 01010001 11111111

该段视频流中 `PTS_DTS_flags == '10'`，故其也有 `5` 字节的 `PTS` 信息，我们尝试解析 `PTS` 信息：

1. `PTS` 的 `5` 字节信息为 `21 00 07 D8 61`，对应二进制 `00100001 00000000 00000111 11011000 01100001`
2. 除去 `reserved` 和 `marker` 字段为 `000 00000000 0000011 11011000 0110000`
3. 整理可得 `0 00000000 00000001 11101100 00110000`，即 `0x001EC30`
4. 转为十进制为 `126000`，与工具计算结果一致

`PES` 包头数据分析完，剩下的数据就音频数据的一部分了。

- [1] [音视频封装：MPTG2-TS 媒体封装实例解析和说明 - 云 + 社区 - 腾讯云](https://cloud.tencent.com/developer/article/1746983)
- [2] {% post_link "音视频封装 - TS 封装格式" %}
- [3] [daniep01/MPEG-2-Transport-Stream-Packet-Analyser: MPEG-2 Transport Stream packet analyser...](https://github.com/daniep01/MPEG-2-Transport-Stream-Packet-Analyser)
- [4] 文中 `TS` 码流视频源文件：[点击下载](/slave/TS/TestVideoTS.7z "密码：github.com")
