---
title: 音视频封装 - TS 封装格式
categories:
  - 音视频封装
tags:
  - MPEG2-TS
abbrlink: c6ae0d94
date: 2020-11-27 14:47:00
---
`TS` 全称是 `MPEG2-TS`，`MPEG2-TS` 是一种标准容器格式，传输与存储音视频、节目与系统信息协议数据，广泛应用于数字广播系统，我们日常数字机顶盒接收到的就是 `TS`（`Transport Stream`，传输流）流。

<!-- more -->

## 概念

首先需要先分辨 `TS` 传输流中几个基本概念：

- `ES` (`Elementary Stream`)：基本流，直接从编码器出来的数据流，可以是编码过的音频、视频或其他连续码流
- `PES` (`Packetized Elementary Streams`)：`PES` 流是 `ES` 流经过 `PES` 打包器处理后形成的数据流，在这个过程中完成了将 `ES` 流分组、加入包头信息（`PTS`、`DTS` 等）操作。`PES` 流的基本单位是 `PES` 包，`PES` 包由包头和 payload 组成
- `PS` 流 (`Program Stream`)：节目流，PS 流由 PS 包组成，而一个 PS 包又由若干个 `PES` 包组成。一个 PS 包由具有同一时间基准的一个或多个 `PES` 包复合合成。
- `TS` 流 (`Transport Stream`)：传输流，`TS` 流由固定长度（`188` 字节）的 `TS` 包组成，`TS` 包是对 `PES` 包的另一种封装方式，同样由具有同一时间基准的一个或多个 `PES` 包复合合成。PS 包是不固定长度，而 `TS` 包为固定长度。

## TS 文件

`TS` 文件分为三次：`TS` 层、`PES` 层、`ES` 层。`ES` 层就是音视频数据，`PES` 层是在音视频数据上加了时间戳等数据帧的说明信息，`TS` 层是在 `PES` 层上加入了数据流识别和传输的必要信息。

    +-+-+-+-+     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  TS   |  =  |  Packet 1 |  Packet 2 |  Packet 3 |    ...    | Packet n-1|  Packet n |
    +-+-+-+-+     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    One Packet:          4bytes              184bytes
    +-+-+-+-+-+    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Packet  | =  | Packet header |       Packet data       |
    +-+-+-+-+-+    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

## TS 层

`TS` 包大小固定为 `188` 字节，`TS` 层分为三个部分：`TS header`、`adaptation field`、`payload`：

- `TS header` 固定 `4` 个字节
- `adaptation field` 可能存在也可能不存在，主要作用是给不足 `188` 字节的数据做填充
- `payload` 是 `PES` 数据

    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | TS header |  adaptation field |          payload(PES)       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

### TS header

    |0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1|
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |       ①       |②|③|④|             ⑤           | ⑥ | ⑦ |   ⑧   |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

| 序号 | 名称 | 大小 | 说明 |
| - | - | - | - |
| 1 | sync_byte | 8bit | 同步字节，固定为 0x47
| 2 | transport_error_indicator | 1bit | 传输错误指示符，表明在 TS 头的 adapt 域后由一个无用字节，通常都为 0，这个字节算在 adapt 域长度内
| 3 | payload_unit_start_indicator | 1bit | 负载单元起始标示符，一个完整的数据包开始时标记为 1
| 4 | transport_priority | 1bit | 传输优先级，0 为低优先级，1 为高优先级，通常取 0
| 5 | PID | 13bit | PID 值
| 6 | transport_scrambling_control | 2bit | 传输加扰控制，00 表示未加密
| 7 | adaptation_field_control | 2bit | 是否包含自适应区，'00'保留；'01'为无自适应域，仅含有效负载；'10'为仅含自适应域，无有效负载；'11'为同时带有自适应域和有效负载。
| 8 | continuity_counter | 4bit | 递增计数器，从 0-f，起始值不一定取 0，但必须是连续的

`TS` 层的内容是通过 `PID` 值来标识的，主要内容包括：`PAT` 表、`PMT` 表、音频流、视频流。解析 `TS` 流要先找到 `PAT` 表，只要找到 `PAT` 就可以找到 `PMT`，然后就可以找到音视频流了。`PAT` 表和 `PMT` 表需要定期插入 `TS` 流，因为用户随时可能加入 `TS` 流，这个间隔比较小，通常每隔几个视频帧就要加入 `PAT` 和 `PMT`。`PAT` 和 `PMT` 表是必须的，还可以加入其它表如 `SDT`（业务描述表）等，不过 `HLS` 流只要有 `PAT` 和 `PMT` 就可以播放了。

- `PAT` 表：他主要的作用就是指明了 `PMT` 表的 `PID` 值。
- `PMT` 表：他主要的作用就是指明了音视频流的 `PID` 值。
- 音频流 / 视频流：承载音视频内容。

`PID` 是 `TS` 流中唯一识别标志，`Packet Data` 是什么内容就是由 `PID` 决定的。下表给出了一些表的 PID 值：

| 表类型 | `PID` 值 |
| - | - |
| PAT | 0x0000 |
| CAT | 0x0001 |
| TSDT | 0x0002 |
| EIT,ST | 0x0012 |
| RST,ST | 0x0013 |

### adaptation field

| 名称 | 大小 | 说明 |
| - | - | - |
| adaptation_field_length | 8bit | 自适应域长度，后面的字节数（从 flag 开始） |
| flag | 8bit | 取 0x50 表示包含 PCR 或 0x40 表示不包含 PCR |
| PCR | 40bit | Program Clock Reference，节目时钟参考，用于恢复出与编码端一致的系统时序时钟 STC（System Time Clock）。 |
| stuffing_bytes | xbit | 填充字节，取值 0xFF |

自适应区的长度要包含传输错误指示符标识的一个字节。`PCR` 是节目时钟参考，`PCR`、`DTS`、`PTS` 都是对同一个系统时钟的采样值，pcr 是递增的，因此可以将其设置为 `DTS` 值，音频数据不需要 `PCR`。如果没有字段，打包 `TS` 流时 `PAT` 和 `PMT` 表是没有 `adaptation field` 的，不够的长度直接补 `0xFF` 即可。视频流和音频流都需要加 `adaptation field`，通常加在一个帧的第一个 `TS` 包和最后一个 `TS` 包里，中间的 `TS` 包不加。

### PAT

`Program Association Table` 节目关联表，每个 `TS` 流对应一张，用来描述该 `TS` 流中有多少个节目。

- `TS` 流中中，`PAT` 包重复实现，大约 `0.5` 秒出现一个，保证实时解码性
- 表示 `PAT` 表的 `TS` 包 `PID` 值为 `0`，便于识别
- `PAT` 的 payload 中传送特殊 `PID` 的列表，每个 `PID` 对应一个节目（对应一张 `PMT` 表）
- `PAT` 表是 `TS` 流的基础，任何一个 `TS` 流解析寻找节目都是从 `PAT` 表开始查找

``` c++
typedef struct TS_PAT_Program {
    unsigned program_number       :16;  // 节目号，为 0x0000 时表示这是 NIT，节目号为 0x0001 时, 表示这是 PMT
    unsigned reserved             :3    // 保留，固定为 111
    unsigned program_map_PID      :13;  // 节目号对应内容的 PID 值
} TS_PAT_Program;

typedef struct TS_PAT {
    unsigned table_id                     : 8;  // 固定 0x00 ，标志是该表是 PAT
    unsigned section_syntax_indicator     : 1;  // 段语法标志位，固定为 1
    unsigned zero                         : 1;  // 0
    unsigned reserved_1                   : 2;  // 保留位，固定为 11
    unsigned section_length               : 12; // 段长度，表示从下一个字段开始到 CRC32(含) 之间有用的字节数
    unsigned transport_stream_id          : 16; // TS 流 ID，一般为 0x0001，区别于一个网络中其它多路复用的流
    unsigned reserved_2                   : 2;  // 保留位，固定为 11
    unsigned version_number               : 5;  // PAT 版本号，固定为 00000，如果 PAT 有变化则版本号加 1
    unsigned current_next_indicator       : 1;  // 固定为 1，表示这个 PAT 表有效，如果为 0 则要等待下一个 PAT 表
    unsigned section_number               : 8;  // 分段的号码。PAT 可能分为多段传输，第一段为 00，以后每个分段加 1，最多可能有 256 个分段
    unsigned last_section_number          : 8;  // 最后一个分段的号码
    std::vector<TS_PAT_Program> program;
    unsigned CRC_32                       : 32; // CRC32 校验码
} TS_PAT;
```

### PMT

`Program Map Table`，节目映射表，该表的 `PID` 是由 `PAT` 表 提供给出的。表征一路节目所有流信息。包含：

1. 当前节目中包含的所有 `Video` 数据的 `PID`
2. 当前节目中包含的所有 `Audio` 数据的 `PID`
3. 与当前节目关联在一起的其他数据的 `PID`（如数字广播, 数据通讯等使用的 `PID`）

如果 `TS` 流中包含多个节目，那么就会有多个 `PMT` 表。只要我们处理了 `PMT` 表，那么我们就可以获取该节目中所有的流信息，如当前节目包含多少个 Video、多少个 `Audio` 和其他数据及每种数据对用的流 `PID` 分别是多少。

``` c++
typedef struct TS_PMT_Stream {
    unsigned stream_type              : 8;  // 指示本节目流的类型，H.264 编码对应 0x1b，AAC 编码对应 0x0f，MP3 编码对应 0x03
    unsigned reserved1                : 3;  // 保留位，固定为 111
    unsigned elementary_PID           : 13; // 指示该流的 PID 值
    unsigned reserved2                : 3;  // 保留位，固定为 1111
    unsigned ES_info_length           : 12; // 前两位 bit 为 00，指示跟随其后的描述相关节目元素的字节数
    std::vector<unsigned> descriptor;
} TS_PMT_Stream;

typedef struct TS_PMT {
    unsigned table_id                     : 8;  // 取值随意，一般使用 0x02, 表示 PMT 表
    unsigned section_syntax_indicator     : 1;  // 段语法标志位，固定为 1
    unsigned zero                         : 1;  // 固定为 0
    unsigned reserved_1                   : 2;  // 保留位，固定为 11
    unsigned section_length               : 12; // 段长度，表示从下一个字段开始到 CRC32(含) 之间有用的字节数
    unsigned program_number               : 16; // 当前 PMT 表映射到的节目号，1、2、3
    unsigned reserved_2                   : 2;  // 保留位，固定为 11
    unsigned version_number               : 5;  // PMT 版本号码，固定为 00000，如果 PAT 有变化则版本号加 1
    unsigned current_next_indicator       : 1;  // 发送的 PMT 表 是当前有效还是下一个 PMT 有效
    unsigned section_number               : 8;  // 分段的号码。PMT 可能分为多段传输，第一段为 00，以后每个分段加 1，最多可能有 256 个分段
    unsigned last_section_number          : 8;  // 分段数
    unsigned reserved_3                   : 3;  // 保留位，固定为 111
    unsigned PCR_PID                      : 13; // 指明 TS 包的 PID 值，该 TS 包含有 PCR 同步时钟，
    unsigned reserved_4                   : 4;  // 预留位，固定为 1111
    unsigned program_info_length          : 12; // 前 2bit 为 00，该域指出跟随其后对节目信息的描述的字节数。
    std::vector<TS_PMT_Stream> PMT_Stream;
    unsigned reserved_5                   : 3;  // 保留位，0x07
    unsigned reserved_6                   : 4;  // 保留位，0x0F
    unsigned CRC_32                       : 32; // CRC32 校验码
} TS_PMT;
```
