---
title: 音视频封装 - MP4 封装格式
categories:
  - 音视频封装
tags:
  - MP4
abbrlink: e90af351
date: 2020-11-29 15:39:12
---
`mp4` 文件封装格式，对应的标准为 `ISO/IEC 14496-12` 和 `ISO/IEC 14496-14`。

`ISO/IEC 14496-12` 定义了一种封装媒体数据的基础文件格式，`mp4`、`3gp`、`ismv` 等我们常见媒体封装格式都是以这种基础文件格式为基础衍生的。

`ISO/IEC 14496-14` 基于 `ISO-14496-12` 定义了 `mp4` 文件格式。

<!-- more -->

## 基础文件格式 - Box

按照 `ISO-14496-12`，文件由一系列对象 `Box` 构成, `Box` 可以嵌套包含其他 `Box`

`Box` 分为两类: 一类是元数据 `Box`，一类是媒体数据 `Box`

![Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201130213738.png)

`Box Header` 的格式可用如下代码表示:

``` c++
aligned(8) class Box (unsigned int(32) boxtype, optional unsigned int(8)[16] extended_type) {
    unsigned int(32) size;
    unsigned int(32) type = boxtype;
    if (size == 1) {
        unsigned int(64) largesize;
    } else if (size == 0) {
        // box extends to end of file
    }
    if (boxtype == "uuid") {
        unsigned int(8)[16] usertype = extended_type;
    }
}
```

一些 `Box` 对象的 `Box Header` 可能会包含有版本号和标志域:

``` c++
aligned(8) class FullBox(unsigned int(32) boxtype, unsigned int(8) v, bit(24) f) extends Box(boxtype) {
    unsigned int(8) version = v;
    bit(24) flags = f;
}
```

## MP4 格式介绍

`Box` 种类非常多，下图中例举了一些重要的 `Box` 类型:

![MP4](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201130214743.png)

1. 分析时会从外到里进行分析，先分析第一层的 `Box` 各个字段，再分析第二层 `Box` 各个字段，以此类推
2. `Box` 由于种类非常多，这里只分析一个标准的 `mp4` 文件的 `Box` 含义
3. `Box` 里面字段也不是所有的都需要关注，我们只需要关注核心和有用的，对于一些不太用的就可以忽略不计了

## 第一层

使用 `mp4info` 打开一个 `mp4` 文件，可以看到，该 `mp4` 文件第一层 `Box` 有 `4` 种:

![第一层](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201130215440.png)

1. `ftyp`
2. `free`
3. `mdat`
4. `moov`

我们逐个分析该 `4` 种 `Box` 的含义和内容。

### ftyp

`ftyp` 是 `mp4` 文件的第一个 `Box`，包含了视频文件使用的编码格式、标准等，这个 `Box` 作用基本就是 `mp4` 这种封装格式的标识，同时在一份 `mp4` 文件中只有一个这样的 `Box`。`ftyp box` 通常放在文件的开始，通过对该 `Box` 解析可以让我们的软件（播放器、demux、解析器）知道应该使用哪种协议对这该文件解析，是后续解读文件基础。

分析如下码流:

![ftyp box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201130234447.png)

使用代码描述其结构:

``` c++
aligned(8) class FileTypeBox extends Box("ftyp") {
    unsigned int(32) major_brand;           // 不同厂商要实现这种规范都要向 ISO 注册的一个四字节的标识码
    unsigned int(32) minor_version;         // 商标版本号
    unsigned int(32) compatible_brands[];   // 兼容其他的版本号，表示可以兼容和遵从那些协议标准，是一个列表
}
```

- Box Header
    - Box Length: `0x00 00 00 1C` 表示该 `Box` 长度为 `28` 字节
    - Box type: `0x66 74 79 70` 是 `ftyp` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data
    - major brand: `0x69 73 6F 6D` 是 `isom` 的 `ASCII` 值，说明本文件是符合这个规范的。
    - minor version: `0x00 00 02 00` 代表 `isom` 的版本号
    - compatible brand: `0x69 73 6F 6D 69 73 6F 32 6D 70 34 31` `ASCII` 值为 `iosmios2mp4l` 表示本文件可以兼容的协议和标准

通过 `Mp4Explorer` 验证分析结果:

![ftyp Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201130235834.png)

### moov

- `Container`: `File`
- `Mandatory`: `Yes`
- `Quantity`: `Exactly one`

`moov Box` 这个 `Box` 也是 `MP4` 文件中必须有但是只存在一个的 `Box`, 这个 `Box` 里面一般存的是媒体文件的元数据，这个 `Box` 本身是很简单的，是一种 Container `Box`，里面的数据是子 `Box`, 自己更像是一个分界标识。

所谓的媒体元数据主要包含类似 `SPS`、`PPS` 的编解码参数信息，还有音视频的时间戳等信息。对于 `MP4` 还有一个重要的采样表 `stbl` 信息，这里面定义了采样 `Sample`、`Chunk`、`Track` 的映射关系，是 `MP4` 能够进行随机拖动和播放的关键，也是需要好好理解的部分，对于实现一些音视频特殊操作很有帮助。

根据 `ISO-14496-12`，文件中必须要包括唯一一个元数据容器: `Movie Box(moov)`，该 `Box` 一般在文件的头部或尾部，方便定位。

分析如下码流:

![moov box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201000619.png)

``` c++
aligned(8) class MovieBox extends Box("moov") { }
```

- Box Header
    - Box Length: `0x00 00 2E 49` 表示该 `Box` 长度为 `11849` 字节
    - Box type: `0x6D 6F 6F 76` 就是 `moov` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data
    - 参见: [第二层 - mvhd](#mvhd)
    - 参见: [第二层 - iods](#iods)
    - 参见: [第二层 - trak](#trak)

通过 `Mp4Explorer` 验证分析结果:

![moov Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201000000.png)

### mdat

`mdat Box` 这个 `Box` 是存储音视频数据的 `Box`，要从这个 `Box` 解封装出真实的媒体数据。当然这个 `Box` 一般都会存在，但是不是必须的。

说明:

1. `mdat Box` 基本组成还是有头部和数据两部分组成，但是这里注意如果 `Box length` 长度不够时，要用后面 `8` 字节的扩展长度字段，`Box Type` 还是 `mdat` 的 `ASCII` 码值
2. 真实的数据字段是一个个 `NALU header` + `NALU Data`，但是没有用 `NALU` 的分解符 `00 00 00 01`, 是在每个 `NALU` 前面加了长度字段，和 `RTP` 打包类似
3. 这里的 `NLAU` 一般不再包含 `SPS` `PPS` 等数据，这些数据已经放到 `moov Box` 里面了，至于是如何放到 `moov Box` 的，下面文章会讲解。这里一般 `NALU` 类型就是 `I\B\P` 帧数据以及 `SEI` 用户增强信息

分析如下码流:

![mdat box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201001516.png)

- Box Header
    - Box Length: `0x00 25 4B F7` 表示该 `Box` 长度为 `2444279` 字节，即后面的 `NALU` 整个长度为 `2444279 - 8 = 2444271` 字节
    - Box type: `0x6D 64 61 74` 这就是 `mdat` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data

### free

`free Box` 中的内容是无关紧要的，可以被忽略即该 `Box` 被删除后，不会对播放产生任何影响。这种类型的 `Box` 也不是必须的，可有可无，类似的 `Box` 还有 `sikp Box`. 虽然在解析是可以忽略，但是需要注意该 `Box` 的删除对其它 `Box` 的偏移量影响，特别是当 `moov Box` 放到 `mdat Box` 后面的情况。

![free box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201222436.png)

- Box Header
    - Box Length: `0x00 00 00 08` 表示该 `Box` 长度为 `9` 字节
    - Box type: `0x66 72 65 65` 这就是 `free` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data

## 第二层

从第二层开始，后续的每一层基本都封装于 `moov Box`。

![moov Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201224431.png)

### mvhd

这个 Box 也是全文件唯一的一个 Box, 一般处于 moov Box 的第一个子 Box, 这个 Box 对整个媒体文件所包含的媒体数据（包含 Video track 和 Audio Track 等）进行全面的描述。其中包含了媒体的创建和修改时间，默认音量、色域、时长等信息。

分析如下码流:

![mvhd Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201231209.png)

使用代码描述其结构:

``` c++
aligned(8) class MovieHeaderBox extends FullBox("mvhd", version, 0) {
    if (version==1) {
        unsigned int(64) creation_time;     // 创建时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(64) modification_time; // 修改时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) timescale;         // 文件媒体在 1 秒时间内的刻度值，可以理解为 1 秒长度的时间单元数，这个只用来计算了该 Mp4 文件的长度，但是没有参与 PTS 和 DTS 的计算。
        unsigned int(64) duration;          // 该影片的播放时长，该值除以 time scale 字段即可以得到该影片的总时长单位秒 s
    } else { // version==0
        unsigned int(32) creation_time;     // 创建时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) modification_time; // 修改时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) timescale;         // 文件媒体在 1 秒时间内的刻度值，可以理解为 1 秒长度的时间单元数，这个只用来计算了该 Mp4 文件的长度，但是没有参与 PTS 和 DTS 的计算。
        unsigned int(32) duration;          // 该影片的播放时长，该值除以 time scale 字段即可以得到该影片的总时长单位秒 s
    }
    template int(32) rate = 0x00010000; // 推荐播放速率，[16.16] 格式，1.0 表示 1 倍速播放
    template int(16) volume = 0x0100;   // 与 rate 类似，[8.8] 格式，1.0 表示最大音量
    const bit(16) reserved = 0;
    const unsigned int(32)[2] reserved = 0;
    template int(32)[9] matrix =
    {0x00010000,0,0,0,0x00010000,0,0,0,0x40000000}; // Unity matrix
    bit(32)[6] pre_defined = 0;     // 预览相关的信息，一般都是填充 0，所以不用太关心
    unsigned int(32) next_track_ID; // 下一个 Track 使用的 id 号，通过该值减去 1 可以判断当前文件的 Track 数量
}
```

- Box Header
    - Box Length: `0x00 00 00 6c` 表示该 `Box` 长度为 `108` 字节
    - Box type: `0x6d 76 68 64` 这就是 `mvhd` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - creation time: `0x00 00 00 00`
    - modification time: `0x00 00 00 00`
    - time_scale: `0x00 00 03 E8` 代表 `1` 秒的时间单位是 `1000`，即把 `1` 秒划分为 `1000` 份，这样描述的更精确
    - duration: `0x00 00 46 68` 这样我们换算下该段 `mp4` 文件的时长是 `18024/1000` 即 `18.024` 秒
    - rate: `0x00 01 00 00` 推荐采样原始倍速播放，一般就是 `1` 倍速进行播放该视频
    - volume: `01 00` 推荐采样原始视频音量进行播放
    - next_track_id: `0x00 00 00 03` 这个值表示如果要增加下一个 `Track` 时，需要的编号是 `3`，那同时也就说明本文件里面有 `2` 个 `Track`，实际发现刚好是 `2` 个 `Track`

通过 `Mp4Explorer` 验证分析结果:

![mvhd Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201224755.png)

### iods

这个 `Box` 也是非必须 `Box`，不算核心 `Box`, 实际也是 `24` 字节的固定值，解析时直接跳过即可。封装直接给填写固定 `24` 字节即可，注意该 `Box` 也是 Full `Box` 意味这 `Header` 里面有 `1` 字节的 `Version` 和 `3` 字节的 `Flag` 字段。

定义的内容应该是 `Audio` 和 `Video ProfileLevel` 方面的描述。但是现在没有用。

### trak

`trak Box` 定义了媒体文件中媒体中一个 `Track` 的信息，视频有 `Video Track`, 音频有 `Audio Track`，媒体文件中可以有多个 `Track`，每个 `Track` 具有自己独立的时间和空间的信息，可以进行独立操作。

每个 `trak Box` 都需要有一个 `tkhd Box` 和 `mdia Box`，其它的 `Box` 都是可选择的

![trak Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201233103.png)

`Trak Box` 这里自身只是一个分界符，属于 `Container Box`，它的 `Box Data` 其实是其他的 `Box` 类型。

- Box Header
    - Box Length: `0x00 00 10 1E` 表示该 `Box` 长度为 `4126` 字节
    - Box type: `0x74 72 61 6B` 这就是 `trak` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data
    - 参见: [第三层 - tkhd](#tkhd)
    - 参见: [第三层 - edts](#edts)
    - 参见: [第三层 - mdia](#mdia)

## 第三层

### tkhd

`Container`: `Track Box` (`"trak"`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one`

该 `Box` 描述了该 `Track` 的媒体整体信息包括时长、图像的宽度和高度等，实际比较重要，同时该 `Box` 是 `Full Box` 即 `Box Header` 后面有 `Version` 和 `Flag` 字段。

分析如下码流:

![tkhd Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201234750.png)

使用代码描述其结构:

``` c++
aligned(8) class TrackHeaderBox extends FullBox(‘tkhd’, version, flags){
    if (version==1) {
        unsigned int(64) creation_time;     // 创建时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(64) modification_time; // 修改时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) track_ID;          // 唯一标识该 Track 的一个非零值
        const unsigned int(32) reserved = 0;
        unsigned int(64) duration;          // 该影片的播放时长，该值除以 time scale 字段即可以得到该影片的总时长单位秒 s
    } else { // version==0
        unsigned int(32) creation_time;     // 创建时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) modification_time; // 修改时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) track_ID;          // 唯一标识该 Track 的一个非零值
        const unsigned int(32) reserved = 0;
        unsigned int(32) duration;          // 该影片的播放时长，该值除以 time scale 字段即可以得到该影片的总时长单位秒 s
    }
    const unsigned int(32)[2] reserved = 0;
    template int(16) layer = 0;             // 视频层，值小的在上层
    template int(16) alternate_group = 0;   // Track 分组信息，一般默认为 0，表示该 Track 和其它 Track 没有建立群组关系
    template int(16) volume = {if track_is_audio 0x0100 else 0}; // [8.8] 格式，1.0 表示最大音量
    const unsigned int(16) reserved = 0;
    template int(32)[9] matrix=
    {0x00010000,0,0,0,0x00010000,0,0,0,0x40000000};
    // unity matrix
    unsigned int(32) width;     // 如果该 Track 为 Video Track，则表示图像的宽度（16.16 浮点表示）
    unsigned int(32) height;    // 如果该 Track 为 Video Track，则表示图像的高度（16.16 浮点表示）
}
```

- Box Header
    - Box Length: `0x00 00 00 5c` 表示该 `Box` 长度为 `92` 字节
    - Box type: `0x74 6B 68 64` 这就是 `tkhd` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 03`
- Box Data
    - creation time: `0x00 00 00 00`
    - modification time: `0x00 00 00 00`
    - track_id: `0x00 00 00 01` 这是该 `Track` 的唯一标识
    - duration: `0x00 00 46 50` 这样我们换算下该段 `mp4` 文件的时长是 `18000/1000` 即 `18` 秒
    - layer: `0x00 00`
    - alternate_group: `0x00 00`
    - volume: `00 00`
    - width: `0x03 55 55 55` 表示该视频图像的宽度是 `853.21845`
    - height: `0x01 e0 00 00` 表示该视频图像的宽度是 `480.0`

通过 `Mp4Explorer` 验证分析结果:

![tkhd Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201201235125.png)

### edts

### mdia

- `Container`: `Track Box` (`trak`)
- `Mandatory`: `Yes`
- `Quantity`: `Exactly one`

这个 `Box` 也是 `Container Box`，里面包含子 `Box`，一般必须有 `mdhd Box`、`hdlr Box`、`minf Box`。基本就是当前 `Track` 媒体头信息和媒体句柄以及媒体信息。

![mdia Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202001301.png)

使用代码描述其结构:

``` c++
aligned(8) class MediaBox extends Box("mdia") { }
```

- Box Header
    - Box Length: `0x00 00 10 1E` 表示该 `Box` 长度为 `4126` 字节
    - Box type: `0x74 72 61 6B` 这就是 `trak` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data
    - 参见: [第四层 - mdhd](#hdlr)
    - 参见: [第四层 - hdlr](#hdlr)
    - 参见: [第四层 - minf](#minf)

## 第四层

### mdhd

- `Container`: `Media Box` (`mdia`)
- `Mandatory`: `Yes`
- `Quantity`: `Exactly one`

这个 `Box` 是 `Full Box`，意味这 `Box Header` 有 `Version` 和 `Flag` 字段，该 `Box` 里面主要定义了该 `Track` 的媒体头信息，其中我们最关心的两个字段是 `Time scale` 和 `Duration`，分别表示了该 `Track` 的时间戳和时长信息，这个时间戳信息也是 `PTS` 和 `DTS` 的单位。

![mdhd Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202001857.png)

使用代码描述其结构:

``` c++
aligned(8) class MediaHeaderBox extends FullBox("mdhd", version, 0) {
    if (version==1) {
        unsigned int(64) creation_time;     // 当前 Track 创建时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(64) modification_time; // 当前 Track 修改时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) timescale;         // 当前 Track 的时间计算单位
        unsigned int(64) duration;          // 当前 Track 的播放时长，需要参考前面的 timescale 计算
    } else { // version==0
        unsigned int(32) creation_time;     // 当前 Track 创建时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) modification_time; // 当前 Track 修改时间（相对于 UTC 时间 1904-01-01 零点的秒数）
        unsigned int(32) timescale;         // 当前 Track 的时间计算单位
        unsigned int(32) duration;          // 当前 Track 的播放时长，需要参考前面的 timescale 计算
    }
    bit(1) pad = 0;
    unsigned int(5)[3] language;            // 媒体语言码，参考 ISO-639-2/T
    unsigned int(16) pre_defined = 0;
}
```

- Box Header
    - Box Length: `0x00 00 00 20` 表示该 `Box` 长度为 `32` 字节
    - Box type: `0x6d 64 68 64` 这就是 `mdhd` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - creation time: `0x00 00 00 00`
    - modification time: `0x00 00 00 00`
    - timescale: `0x00 00 30 00` 表示当前 `Track` 的时间戳单位是 `12288`
    - duration: `0x00 03 60 00` 这样我们换算下该段 `mp4` 文件的时长是 `221184/12288` 即 `18` 秒
    - language: `0x55 c4` 最高位为 `0`，后面 `15` 位代表三个字符
        - `0x55 c4` 转为二进制 `BIN: 01010101 11000100` 去掉首位 `0` 得到 `15` 位二进制数字
        - 每 `5` 位二进制一组: `10101 01110 00100` 代表三个数字 `21`、`15`、`4`
        - 三个数字对应从 `a` 开始的第 `21`、`15`、`4` 个字母，即 `u`、`n`、`d`，查询 [ISO 639-2/T](https://en.wikipedia.org/wiki/List_of_ISO_639-2_codes) 可知其含义
    - qualiiy: `0x00 00`

通过 `Mp4Explorer` 验证分析结果:

![mdhd Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202001924.png)

### hdlr

`Container`: `Media Box` (`mdia`) or `Meta Box` (`meta`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one`

这个 `Box` 是 `Full Box`，该 `Box` 解释了媒体的播放过程信息，用来设置不同 `Track` 的处理方式，标识了该 `Track` 的类型。

![hdlr Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202003746.png)

使用代码描述其结构:

``` c++
aligned(8) class HandlerBox extends FullBox("hdlr", version = 0, 0) {
    unsigned int(32) pre_defined = 0;
    unsigned int(32) handler_type;  // Handle 的类型: 视频: 'vide'，音频: 'soun'
    const unsigned int(32)[3] reserved = 0;
    string name; // 这个 component 的名字，其实这里就是你给该 Track 其的名字，打包时填写一个有意义字符串就可以
}
```

- Box Header
    - Box Length: `0x00 00 00 2d` 表示该 `Box` 长度为 `45` 字节
    - Box type: `0x68 64 6c 72` 这就是 `hdlr` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - handle type: `0x76 69 64 65` 即'vide'，代表当前 Track 为视频数据
    - name: `VideoHandler`

通过 `Mp4Explorer` 验证分析结果:

![hdlr Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202003621.png)

### minf

`Container`: `Media Box` (`mdia`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one`

这个 `Box` 是我认为 moov `Box` 里面最重要最复杂的 `Box`，内部还有子 `Box`。该 `Box` 建立了时间到真实音视频 `Sample` 的映射关系，对于音视频数据操作时很有帮助的。

同时该 `Box` 是 `Container Box`，下面一般含有三大必须的子 `Box`:

- 媒体信息头 `Box`: `vmhd Box` 或者 `smhd Box`;
- 数据信息 `Box`: `dinf Box`
- 采样表 `Box`:`stbl Box`

![minf Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202124113.png)

使用代码描述其结构:

``` c++
aligned(8) class MediaInformationBox extends Box("minf") { }
```

- Box Header
    - Box Length: `0x00 00 0f 41` 表示该 `Box` 长度为 `3905` 字节
    - Box type: `0x6d 69 6e 66` 这就是 `trak` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data
    - 参见: [第五层 - vmhd](#vmhd)
    - 参见: [第五层 - smhd](#smhd)
    - 参见: [第五层 - dinf](#dinf)
    - 参见: [第五层 - stbl](#stbl)

## 第五层

### vmhd

- `Container`: `Media Information Box` (`minf`)
- `Mandatory`: `Yes`
- `Quantity`: `Exactly one`

这个 `Box` 的大小是固定的，大小是 `24` 字节，一般都是用默认值填充。

![vmhd Box](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202125231.png)

使用代码描述其结构:

``` c++
aligned(8) class VideoMediaHeaderBox extends FullBox("vmhd", version = 0, 1) {
    template unsigned int(16) graphicsmode = 0; // 视频合成模式，为 0 时拷贝原始图像，否则与 opcolor 进行合成
    template unsigned int(16)[3] opcolor = {0, 0, 0}; // 颜色值，RGB 颜色值，{R,G,B} 一般默认 0x00 00 00 00 00 00
}
```

### smhd

- `Container`: `Media Information Box` (`minf`)
- `Mandatory`: `Yes`
- `Quantity`: `Exactly one specific media header shall be present`

这个 `Box` 的大小是固定的，大小是 `16` 字节，一般都是用默认值填充。

使用代码描述其结构:

``` c++
aligned(8) class SoundMediaHeaderBox extends FullBox("smhd", version = 0, 0) {
    template int(16) balance = 0;   // 音频的均衡是用来控制计算机的两个扬声器的声音混合效果，一般是 0
    const unsigned int(16) reserved = 0;
}
```

### dinf

`Container`: `Media Information Box` (`minf`) or `Meta Box` (`meta`)
`Mandatory`: `Yes` (required within `minf box`) and `No` (optional within `meta box`)
`Quantity`: `Exactly one`

这个 `Box` 也是一个 `Container Box`，一般用来定位媒体信息。一般会包含一个 `dref Box` 即 `data reference box`，`dref` 下面会有若干个 `url Box` 或者也叫 `urn Box`，这些 `Box` 组成一个表，用来定位 `Track` 的数据。

`Track` 可以被分成若干个段，每一段都可以根据 `Url` 或者 `Urn` 指向的地址来获取数据，`sample` 描述中会用这些片段的序号将这些片段组成一个完整的 `track`，一般情况下当数据完全包含在文件中，`Url` 和 `urn Box` 的字符串是空的。

这个 `Box` 存在的意义就是允许 `MP4` 文件的媒体数据分开最后还能进行恢复合并操作，但是实际上，`Track` 的数据都保存再文件中，所以该字段的重要性还体现不出来。

![dinf](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202130034.png)

使用代码描述其结构:

``` c++
aligned(8) class DataInformationBox extends Box("dinf") { }
```

- Box Header
    - Box Length: `0x00 00 00 24` 表示该 `Box` 长度为 `36` 字节
    - Box type: `0x64 69 6e 66` 这就是 `dinf` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data
    - 参见: [第六层 - dref](#dref)

### stbl

- `Container`: `Media Information Box` (`minf`)
- `Mandatory`: `Yes`
- `Quantity`: `Exactly one`

![stbl](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202215751.png)

stbl Box 同样是 Container Box，使用代码描述其结构:

``` c++
aligned(8) class SampleTableBox extends Box("stbl") { }
```

- Box Header
    - Box Length: `0x00 00 0f 01` 表示该 `Box` 长度为 `3841` 字节
    - Box type: `0x73 74 62 6c` 这就是 `stbl` 的 `ASCII` 值，标识了该 `Box` 的类型
- Box Data
    - 参见: [第六层 - stsd](#stsd)
    - 参见: [第六层 - stts](#stts)
    - 参见: [第六层 - stss](#stss)
    - 参见: [第六层 - stsc](#stsc)
    - 参见: [第六层 - stsz](#stsz)
    - 参见: [第六层 - stco](#stco)

## 第六层

### dref

`Container`: `Data Information Box` (`dinf`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one`

由于 `Track` 可以分为多个段，所以该 `Box` 是用来定义当前 `Track` 各个段的 `URL` 或 `URN`。

![dref](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202130842.png)

使用代码描述其结构:

``` c++
aligned(8) class DataEntryUrlBox (bit(24) flags) extends FullBox("url", version = 0, flags) {
    string location;
}
aligned(8) class DataEntryUrnBox (bit(24) flags) extends FullBox("urn", version = 0, flags) {
    string name;
    string location;
}
aligned(8) class DataReferenceBox extends FullBox("dref", version = 0, 0) {
    unsigned int(32) entry_count;
    for (int i = 1; i <= entry_count; i++) {
        DataEntryBox(entry_version, entry_flags) data_entry;
    }
}
```

- Box Header
    - Box Length: `0x00 00 00 1c` 表示该 `Box` 长度为 `28` 字节
    - Box type: `0x64 72 65 66` 这就是 `dref` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - entry count: `0x00 00 00 01` 说明下面的 `url` 列表数量是 `1`，只有一个 `url Box`
    - URL Box 1
        - Box Length: `0x00 00 00 0c` 表示该 `Box` 长度为 `12` 字节
        - Box type: `0x75 72 6c 20` 这就是 `"url"` 的 `ASCII` 值，标识了该 `Box` 的类型
        - Box version: `0x00`
        - Box flags: `0x00 00 01`

### stsd

`Container`: `Sample Table Box` (`stbl`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one`

该 `Box` 存储了编码类型和初始化解码器需要的信息，于特定的 `track-type` 有关，根于不同的 `Track` 使用不一样的编码标准。

![stsd](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201202220310.png)

使用代码描述其结构:

``` c++
aligned(8) abstract class SampleEntry (unsigned int(32) format) extends Box(format){
    const unsigned int(8)[6] reserved = 0;
    unsigned int(16) data_reference_index;
}
class BitRateBox extends Box("btrt"){
    unsigned int(32) bufferSizeDB;
    unsigned int(32) maxBitrate;
    unsigned int(32) avgBitrate;
}
aligned(8) class SampleDescriptionBox (unsigned int(32) handler_type) extends FullBox("stsd", version, 0){
    unsigned int(32) entry_count;   //sample description 数目, 同时不同的 Track 有不同的 sample description
    for (int i = 1 ; i <= entry_count ; i++){
        SampleEntry(); // an instance of a class derived from SampleEntry
    }
}
```

- Box Header
    - Box Length: `0x00 00 01 09` 表示该 `Box` 长度为 `265` 字节
    - Box type: `0x73 74 73 64` 这就是 `stsd` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - entry count: `0x00 00 00 01` 说明下面的 `Sample Description Box` 列表数量是 `1`
    - Sample Description Box: 参见文档 ISO-IEC 14496-15 AVC file format

### stts

`Container`: `Sample Table Box` (`stbl`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one`

这个 Box 是 `Sample number` 和解码时间 `DTS` 之间的映射表，通过这个表格，我们可以找到任何时间的 `Sample`，但用这个表格只是能找到当前时间的 `Sample` 的序号，`Sample` 的长度和指针还要靠其它表格获取。

`Sample delta` 简单可以理解为采样点 `Sample` 的 `duration`，所有 `duration` 相加应该为总的 `Track` 的时长。大家把这个时间理解为该帧数据当前的解码时间即可，计算公式：

    DT(n+1) = DT(n) + STTS(n)

![stts](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201203130420.png)

使用代码描述其结构:

``` c++
aligned(8) class TimeToSampleBox extends FullBox("stts", version = 0, 0) {
    unsigned int(32) entry_count;
    for (int i = 0; i < entry_count; i++) {
        unsigned int(32) sample_count;  // 连续相同时间长度 `sample delta` 的 `sample` 个数
        unsigned int(32) sample_delta;  // 每个 `sample delta` 以 `timescale` 为单位的时间长度
    }
}
```

- Box Header
    - Box Length: `0x00 00 00 18` 表示该 `Box` 长度为 `24` 字节
    - Box type: `0x73 74 74 73` 这就是 `stts` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - entry count: `0x00 00 00 01` 说明下面的 `sample conut` 和 `sample delta` 组成的二元组信息个数 列表数量是 `1`
    - entry 1
        - sample conut: `0x00 00 01 b0` 值为 `sample delta` 的 `sample` 个数为 `432` 个
        - sample delta: `0x00 00 02 00` 值为 `512`，这样我们计算出帧率就是 `12288/512` 即 `24fps`，参见 [第四层 - mdhd](#mdhd)。

通过 `Mp4Explorer` 验证分析结果:

![stts Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201203131946.png)

### ctts

### stss

`Container`: `Sample Table Box` (`stbl`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one`

`I` 帧是播放的起始位置，只有编码器拿到第一个 `I` 帧才能渲染出第一幅画面。所以后续的一些特殊随机操作，高标清切换时都需要找 `I` 帧，只有随机找到 `I` 帧才能完成这些特殊操作。

其中该 `Box` 就是存储了那些 `Sample` 是 `I` 帧，很显然音频 `Track` 也是不存在这个 `Box` 的。

![stss](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201203132044.png)

使用代码描述其结构:

``` c++
aligned(8) class SyncSampleBox extends FullBox("stss", version = 0, 0) {
    unsigned int(32) entry_count;           // 关键帧的 sample 个数
    for (int i = 0; i < entry_count; i++) {
        unsigned int(32) sample_number;     // 关键帧 sample 序号
    }
}
```

- Box Header
    - Box Length: `0x00 00 00 18` 表示该 `Box` 长度为 `24` 字节
    - Box type: `0x73 74 73 73` 这就是 `stss` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - entry count: `0x00 00 00 02` 说明本文件有 `2` 个关键帧
    - sample number 1: `0x00 00 00 01` 关键帧 `1` 的 `Sample` 序号为 `1`
    - sample number 2: `0x00 00 00 fb` 关键帧 `2` 的 `Sample` 序号为 `251`

### stsc

`Container`: `Sample Table Box` (`stbl`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one`

这个 `Box` 就是说明那些 `Sample` 可以划分为一个 `Trunk`

媒体数据被分为若干个 `Chunk`, `Chunk` 可以有不同的大小，同一个 `Chunk` 中的样点 `Sample` 也允许有不同的大小；通过本表可以定位一个样点的 `Chunk` 位置，并可以得知该 `MP4` 中有多少个 `Chunks`，每个 `Chunks` 有多少个 `Samples`。

![stsc](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201203132226.png)

使用代码描述其结构:

``` c++
aligned(8) class SampleToChunkBox extends FullBox("stsc", version = 0, 0) {
    unsigned int(32) entry_count;
    for (int i = 1; i <= entry_count; i++) {
        /* 具有相同采样点 sample 和 `sample_description_index 的 chunk 中，第一个 chunk 的索引值
        ** 也就是说该 chunk 索引值一直到下一个索引值之间的所有 chunk 都具有相同的 sample 个数
        ** 同时这些 sample 的描述 description 也一样； */
        unsigned int(32) first_chunk;
        unsigned int(32) samples_per_chunk;         // 上面所有 chunk 的 sample 个数
        unsigned int(32) sample_description_index;  // 描述采样点的采样描述项的索引值，范围为 1 到样本描述表中的表项数目
    }
}
```

- Box Header
    - Box Length: `0x00 00 00 18` 表示该 `Box` 长度为 `24` 字节
    - Box type: `0x73 74 73 73` 这就是 `stss` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - entry count: `0x00 00 00 01` 这说明三元组信息的个数只有 `1` 个
    - first chunk 1: `0x00 00 00 01` 具有相同 `Sample` 个数和 `description` 的 `Chunk` 起始索引是 `1`
    - samples per chunk 1: `0x00 00 00 01` 每个 `Chunk` 具有的 `Sample` 的个数是 `1`
    - sample description index 1 `0x00 00 00 01` 这些 `Sample` 的 `description index` 是 1，一般用默认值即可

通过 `Mp4Explorer` 验证分析结果:

![stsc Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201203231441.png)

### stsz

`Container`: `Sample Table Box` (`stbl`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one variant must be present`

前面分析了 `Sample` 的 `PTS`、`DTS` 等，也分析了 `Chunk` 里面 `Sample` 的信息，但是没有分析 `Sample` 的大小，这是我们在文件读取和解析 `Sample` 的关键。这里给出每个 `Sample` 的 `Size` 即包含的字节数。

包含了媒体中全部 `Sample` 的数目和一张给出每个 `Sample` 大小的表。这个 `Box` 相对来说体积是比较大的。

![stsz](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201203233016.png)

使用代码描述其结构:

``` c++
aligned(8) class SampleSizeBox extends FullBox("stsz", version = 0, 0) {
    unsigned int(32) sample_size;   // 指定默认的 Sample 大小，如果每个 Sample 大小不相等，则这个字段值为 0，每个 Sample 大小存在下表中
    unsigned int(32) sample_count;  // 该 Track 中所有 Sample 的数量
    if (sample_size == 0) {
        for (int i = 1; i <= sample_count; i++) {
            unsigned int(32) entry_size;    // 每个 Sample 的大小
        }
    }
}
```

- Box Header
    - Box Length: `0x00 00 06 d4` 表示该 `Box` 长度为 `24` 字节
    - Box type: `0x73 74 73 7a` 这就是 `stsz` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - sample size: `0x00 00 00 00` 说明 `Sample` 的具体大小在表后面项目中的 `entry size`
    - sample count: `0x00 00 01 b0` 说明本 `Track` 的大小为 `432` 个 `Sample`
    - entry size 1: `0x00 00 73 5f` 说明第一个 `Sample` 大小为 `29535` 字节
    - entry size 2: `0x00 00 06 e6` 说明第二个 `Sample` 大小为 `1766` 字节
    - entry size 3: `0x00 00 1f be` 说明第三个 `Sample` 大小为 `8126` 字节
    - entry size 4: `0x00 00 1b e2` 说明第四个 `Sample` 大小为 `7138` 字节
    - entry size 5: `0x00 00 25 eb` 说明第五个 `Sample` 大小为 `9707` 字节
    - ...

通过 `Mp4Explorer` 验证分析结果:

![stsz Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201203232256.png)

### stco | co64

`Container`: `Sample Table Box` (`stbl`)
`Mandatory`: `Yes`
`Quantity`: `Exactly one variant must be present`

该 `Box` 存储了 `Chunk Offset`，表示了每个 `Chunk` 在文件中的位置，这样我们就能找到了 `Chunk` 在文件的偏移量，然后根据其它表的关联关系就可以读取每个 `Sample` 的大小。

![stco](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201204000004.png)

使用代码描述其结构:

``` c++
aligned(8) class ChunkOffsetBox extends FullBox("stco", version = 0, 0) {
    unsigned int(32) entry_count;
    for (int i = 1; i <= entry_count; i++) {
        unsigned int(32) chunk_offset;
    }
}

aligned(8) class ChunkLargeOffsetBox extends FullBox("co64", version = 0, 0) {
    unsigned int(32) entry_count;
    for (int i = 1; i <= entry_count; i++) {
        unsigned int(64) chunk_offset;
    }
}
```

- Box Header
    - Box Length: `0x00 00 06 d0` 表示该 `Box` 长度为 `24` 字节
    - Box type: `0x73 74 63 6f` 这就是 `stco` 的 `ASCII` 值，标识了该 `Box` 的类型
    - Box version: `0x00`
    - Box flags: `0x00 00 00`
- Box Data
    - entry count: `0x00 00 01 b0` 说明本 `Track` 的大小为 `432` 个 `Sample`
    - chunk offset 1: `0x00 00 00 43` 说明本文件第一个 `Chunk` 的偏移量是 `67` 字节
    - chunk offset 2: `0x00 00 74 a2` 说明本文件第二个 `Chunk` 的偏移量是 `29858` 字节
    - chunk offset 3: `0x00 00 7e 35` 说明本文件第三个 `Chunk` 的偏移量是 `32309` 字节
    - chunk offset 4: `0x00 00 a0 53` 说明本文件第四个 `Chunk` 的偏移量是 `41043` 字节
    - chunk offset 5: `0x00 00 be 3f` 说明本文件第五个 `Chunk` 的偏移量是 `48703` 字节
    - ...

通过 `Mp4Explorer` 验证分析结果:

![stco Mp4Explorer](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201204000821.png)

## 参考资料

* [1] [ISO/IEC 14496-12](/slave/MP4/ISO-IEC-14496-12-Base-Format-2015.pdf)
* [2] [ISO/IEC 14496-14](/slave/MP4/ISO-IEC-14496-14-MP4-2003.pdf)
