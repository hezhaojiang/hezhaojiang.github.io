---
title: 音视频封装 - FLV 封装格式
categories:
  - 音视频封装
tags:
  - FLV
abbrlink: dff5a5c3
date: 2020-11-15 14:42:37
---
关于 FLV 封装格式，随便在网络上一搜索，便能搜到一张包浆图：

![FLV 封装格式](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201115225122.png)

<!-- more -->

## The FLV header

An FLV file shall begin with the FLV header:

| Field | Type | Comment |
| - | - | - |
| Signature | uint_8 | Signature byte always 'F' (0x46) |
| Signature | uint_8 | Signature byte always 'L' (0x4C) |
| Signature | uint_8 | Signature byte always 'V' (0x56) |
| Version | uint_8 | File version (for example, 0x01 for FLV version 1) |
| TypeFlagsReserved | ubit[5] | Shall be 0 |
| TypeFlagsAudio | ubit[1] | 1 = Audio tags are present |
| TypeFlagsReserved | ubit[1] | Shall be 0 |
| TypeFlagsVideo | ubit[1] | 1 = Video tags are present |
| DataOffset | uint_32 | The length of this header in bytes |

The DataOffset field usually has a value of 9 for FLV version 1. This field is present to accommodate larger headers
in future versions.

## The FLV File Body

After the FLV header, the remainder of an FLV file shall consist of alternating back-pointers and tags. They
interleave as shown in the following table:

FLV File Body

| Field | Type | Comment |
| - | - | - |
| PreviousTagSize0 | uint_32 | Always 0 |
| Tag1 | FLVTAG | First tag |
| PreviousTagSize1 | uint_32 | Size of previous tag, including its header, in bytes. For FLV version1, this value is 11 plus the DataSize of the previous tag. |
| Tag2 | FLVTAG | Second tag |
| ... | ... | ... |
| PreviousTagSizeN-1 | uint_32 | Size of second-to-last tag, including its header, in bytes. |
| TagN | FLVTAG | Last tag |
| PreviousTagSizeN | uint_32 | Size of last tag, including its header, in bytes. |

## FLV Tag Definition

### FLV Tag Header

The FLV tag contains metadata for audio, video, or scriPTS, optional encryption metadata, and the payload.

| Field | Type | Comment |
| - | - | - |
| Reserved | ubit[2] | 保留字段 恒为 0 |
| Filter | ubit[1] | 0：无需预处理 1：需要对包进行预处理（如解密） |
| TagType | ubit[5] | tag 类型 |
| DataSize | uint_24 | 消息长度，包括从 StreamID 以后到 tag 末尾的数据 |
| Timestamp | uint_24 | 当前 tag 的时间戳，以毫秒为单位 |
| TimestampExtended | uint_8 | 时间戳的拓展字节 |
| StreamID | uint_24 | ≡ 0 |

- TagType（tag 类型）
    - 8 = 音频
    - 9 = 视频
    - 18 = 脚本数据

In playback, the time sequencing of FLV tags depends on the FLV timestamps only. Any timing mechanisms built
into the payload data format shall be ignored.

### Audio Tags

FLV 对 AAC 音频格式封装比较特殊，本文中忽略对 AAC 格式音频的 FLV 封装介绍

The AudioTagHeader contains audio-specific metadata.

| Field | Type | Comment |
| - | - | - |
| SoundFormat | ubit[4] | 音频格式 |
| SoundRate | ubit[2] | 音频采样率 |
| SoundSize | ubit[1] | 音频采样大小 |
| SoundType | ubit[1] | 单声道 / 立体声 |
| AACPacketType | IF (SoundFormat == 10) uint_8 |
| AUDIODATA | Varies by SoundFormat | |

- SoundFormat
    - 0 = Linear PCM, platform endian
    - 1 = ADPCM
    - 2 = MP3
    - 3 = Linear PCM, little endian
    - 4 = Nellymoser 16 kHz mono
    - 5 = Nellymoser 8 kHz mono
    - 6 = Nellymoser
    - 7 = G.711 A-law logarithmic PCM
    - 8 = G.711 mu-law logarithmic PCM
    - 9 = reserved
    - 10 = AAC
    - 11 = Speex
    - 14 = MP3 8 kHz
    - 15 = Device-specific sound
- SoundRate
    - 0 = 5.5 kHz
    - 1 = 11 kHz
    - 2 = 22 kHz
    - 3 = 44 kHz
- SoundSize
    - 0 = 8-bit samples
    - 1 = 16-bit samples
- SoundType
    - 0 = Mono sound
    - 1 = Stereo sound
- AACPacketType
    - 0 = AAC sequence header
    - 1 = AAC raw

### Video Tags

| Field | Type | Comment |
| - | - | - |
| Frame Type | ubit[4] | 视频帧类型 |
| CodecID | ubit[4] | 视频编码格式 |
| AVCPacketType | IF (CodecID == 7) uint_8 | AVC 包类型 |
| CompositionTime | IF (CodecID == 7) uint_24 | 单声道 / 立体声 |
| VIDEODATA | VideoTagBody | Video frame payload or frame info |

- Frame Type
    1 = key frame (for AVC, a seekable frame)
    2 = inter frame (for AVC, a non-seekable frame)
    3 = disposable inter frame (H.263 only)
    4 = generated key frame (reserved for server use only)
    5 = video info/command frame
- CodecID
    2 = Sorenson H.263
    3 = Screen video
    4 = On2 VP6
    5 = On2 VP6 with alpha channel
    6 = Screen video version 2
    7 = AVC
- AVCPacketType
    0 = AVC sequence header
    1 = AVC NALU
    2 = AVC end of sequence (lower level NALU sequence ender is not required or supported)
- CompositionTime
    - IF (AVCPacketType == 1) CTS 偏移
    - IF (AVCPacketType == 0) 0

{% note info no-icon %}
关于 CTS：这是一个比较难以理解的概念，需要和 PTS，DTS 配合一起理解。
首先，PTS（presentation time stamps），DTS(decoder timestamps)，CTS(CompositionTime) 的概念：
PTS：显示时间，也就是接收方在显示器显示这帧的时间。单位为 1/90000 秒。
DTS：解码时间，也就是 RTP 包中传输的时间戳，表明解码的顺序。单位单位为 1/90000 秒。根据后面的理解，PTS 就是标准中的 CompositionTime
CTS 偏移：CTS = (PTS - DTS) / 90 。CTS 的单位是毫秒。
PTS 和 DTS 的时间不一样，应该只出现在含有 B 帧的情况下，也就是 main profile 以上。baseline 是没有这个问题的，baseline 的 PTS 和 DTS 一直 x 相同，所以 CTS 一直为 0。
{% endnote %}

### Data Tags

| Field | Type | Comment |
| - | - | - |
| Name | SCRIPTDATAVALUE | Method or object name. SCRIPTDATAVALUE.Type = 2 (String) |
| Value | SCRIPTDATAVALUE | AMF arguments or object properties. SCRIPTDATAVALUE.Type = 8 (ECMA array) |

SCRIPTDATAVALUE 类型类似于 AMF 类型：

| Field | Type | Comment |
| - | - | - |
| Type | uint_8 | Type of the ScriptDataValue. |
| ScriptDataValue | Varies by Type | Script data value. |

每种 Type 的取值对应的 ScriptDataValue 如下：

| Type Value | Type | ScriptDataValue |
| - | - | - |
| 0 | Number | DOUBLE |
| 1 | Boolean | uint_8 |
| 2 | String | SCRIPTDATASTRING |
| 3 | Object | SCRIPTDATAOBJECT |
| 4 | MovieClip (reserved, not supported) | |
| 5 | Null | |
| 6 | Undefined | |
| 7 | Reference | uint_16 |
| 8 | ECMA array | SCRIPTDATAECMAARRAY |
| 9 | Object end marker | |
| 10 | Strict array | SCRIPTDATASTRICTARRAY |
| 11 | Date | SCRIPTDATADATE |
| 12 | Long string | SCRIPTDATALONGSTRING |

- Number: 8 字节 DOUBLE 类型，比如十六进制 `00 40 10 00 00 00 00 00 00` 就表示的是一个 double 数 `4.0`
- Boolean: 1 字节，使用 `00` 表示 `false`，使用 `01` 表示 `true`
- String: 16 位整数表示字符串的长度 + 字符串
- Object: 内容由多组 String 类型的 SCRIPTDATAVALUE 作为 Key，其他类型的 SCRIPTDATAVALUE 作为 Value 的哈希列表组成，由 3 个字节：`00 00 09` 来表示结束
- Null: 仅占用 1 字节，那就是 Null 对象标识 0x05
- ECMA array: 32 位整数表示 Object 对象中元素个数 + Object 类型的 SCRIPTDATAVALUE
- Strict array: 32 位整数表示数组元素个数 + 多个其他类型的 SCRIPTDATAVALUE
- Date: 8 字节 double 来表示从 1970/1/1 到表示的时间所经过的毫秒数 + 2 字节无符号整数表示时区
- Long string: 32 位整数表示字符串的长度 + 字符串

如 FLV 的 Data Tags 信息中最主要的 onMetaData 信息如下：

    02 20 0a 6f 6e 4d 65 74 61 44 61 74 61 08 20 20 20 0d 20 08 64 75 72 61 74 69 6f 6e 20 40 32 0c
    49 ba 5e 35 3f 20 05 77 69 64 74 68 20 40 8a c0 20 20 20 20 20 20 06 68 65 69 67 68 74 20 40 7e
    20 20 20 20 20 20 20 0d 76 69 64 65 6f 64 61 74 61 72 61 74 65 20 40 95 41 68 20 20 20 20 20 09
    66 72 61 6d 65 72 61 74 65 20 40 38 20 20 20 20 20 20 20 0c 76 69 64 65 6f 63 6f 64 65 63 69 64
    20 40 1c 20 20 20 20 20 20 20 0d 61 75 64 69 6f 64 61 74 61 72 61 74 65 20 40 5f 40 20 20 20 20
    20 20 0f 61 75 64 69 6f 73 61 6d 70 6c 65 72 61 74 65 20 40 e5 88 80 20 20 20 20 20 0f 61 75 64
    69 6f 73 61 6d 70 6c 65 73 69 7a 65 20 40 30 20 20 20 20 20 20 20 06 73 74 65 72 65 6f 01 01 20
    0c 61 75 64 69 6f 63 6f 64 65 63 69 64 20 40 24 20 20 20 20 20 20 20 07 65 6e 63 6f 64 65 72 02
    20 0d 4c 61 76 66 35 38 2e 32 39 2e 31 30 30 20 08 66 69 6c 65 73 69 7a 65 20 41 4a 9e 05 80 20
    20 20 20 20 09 20 20 01 30 09

解析后以上数据如下所示：

![onMetaData 信息](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117000710.png)

## 参考资料
