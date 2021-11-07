---
title: 音视频封装 - PS 封装解析示例
categories:
  - 音视频封装
tags:
  - MPEG2-PS
abbrlink: 1472cc3
date: 2020-11-17 13:03:07
---
文章 {% post_link "音视频封装 - PS 封装格式" %} 中分析了 `PS` 流的结构，但文章中中涉及到的元素过多，难以理解，故通过一段真实的码流的分析来加强对 `PS` 流结构的理解。

> 注 1：本文中的 `PS` 码流 A 截取自一段经 `ffmpeg` 转码的网络视频，码流 B 截取自海康摄像机的 `GB28181` 码流。
> 注 2：本文中的数字一般为十进制表示，如果是其他进制数据，会在数字后携带 `(2)` 代表二进制，在数字前添加 `0x` 代表十六进制
> 注 3：截图中的码流信息均使用十六进制

<!-- more -->

## PS Stream A

### PS Header

`PS` 码流由多个 `PS Pack` 组成，而 `PS Pack` 的起始符为 `0x00 00 01 ba`。

![PS Header](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117214228.png)

接下来跳过 `9` 个字节，看第 `10` 个字节，其最后三位数字代表拓展内容的长度，此包中该位置后三位为 `0`，代表无拓展内容。

> 如果该位置后三位非 0，参见 [PS Header Extern](#PS-Header-Extern)

![拓展内容](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117215716.png)

继续读取数据，如果遇到了 `0x00 00 01 bb`，就代表到了码流中 `PS System Header` 部分。

> 码流中有可能会读取到 `0x00 00 01 bc`，参见 [Program Stream Map](#Program-Stream-Map)

### PS System Header

`System Header` 当且仅当数据包为第一个数据包时才存在，以 `0x00 00 01 bb` 的开始。

![PS System Header](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117220401.png)

之后紧跟着的 `0x00 0c` 两个字节表示 `System Header` 的长度，换算为十进制，即为 `12` 个字节。

![PS System Header Length](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117220711.png)

之后跳过 `6` 字节数据，之后如果 `nextbit == 1`，则该位置为一个 `stream_id`，`stream_id` 为 `0xe0`，指示该 `PS` 码流中包含视频流。

![Stream id e0](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117221430.png)

该部分内容 `3` 字节一个循环，下一个循环中 `nextbit == 1`，则该位置也为一个 `stream_id`，`stream_id` 为 `0xc0`，指示该 `PS` 码流中包含音频流。

![Stream id c0](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117221746.png)

至此 `nextbit == 0`，说明该循环结束，读取的长度也与上述 `PS System Header` 的长度值 `12` 一致。

继续读取数据，遇到了 `0x00 00 01`，`PS` 流中这三个字符可视为不同数据的分隔符，而其下一位数据指示了后续数据的类型。

### PES Pack

如果读取 PS 流中遇到了 `0x00 00 01 e0` 或 `0x00 00 01 c0`，就代表读取到了 `PES Pack`。

`PES Pack` 分为两个部分，一部分是 `Header`，一部分是 `Payload`，`Header` 用于存储一些描述信息，而 `Payload` 部分为其存储的原始数据，`PES` 可能有多个。

![PES Pack](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117234659.png)

`0x00 00 01 e0` 代表遇到的是一个 `PES Video Pack`，如果是 `0x00 00 01 c0`，代表遇到的是一个 `PES Audio Pack`。

再之后的 `0x07 dc` 表示长度，即其后的 `2012` 字节为数据内容长度。

![PES Pack Length](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117235040.png)

之后参考 `PES Pack` 结构来解析数据，可参见 {% post_link "音视频封装 - PES 封装格式" %}。

跳过 1 字节字段后，下一个字段为 `PTS_DTS_flags` 字段，占位 2bits。

> `PTS_DTS_flags`：`PTS` 和 `DTS` 标志位，占位 2bit；`10(2)` 表示首部有 `PTS` 字段，`11(2)` 表示有 `PTS` 和 `DTS` 字段，`00(2)` 表示都没有，`01(2)` 被禁止。
> `PTS` - 显示时间戳（Presentation Time Stamp），用来表示显示单元出现在系统目标解码器的时间。
> `DTS` - 解码时间戳（Decoding Time Stamp），用来表示将存取单元全部字节从解码缓存取走的时间。
> 如果 `PTS_DTS_flags` 不是 `00(2)`，就代表存在 `PTS` 或 `DTS` 参见 [PTS_DTS_flags](#PTS_DTS_flags)

故该 `PES Pack` 中无 `PTS` 和 `DTS` 字段

![PTS_DTS_flags](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117235708.png)

后面 6bits 也是 `PES` 包中可选字段的 `Flag` 参数，由上图可知，该 `PES Pack` 无这些可选字段。

继续读取数据，下一个字段就为 `PES_header_data_length`，代表 `PES Header` 中可选字段和填充字段的长度，占位 `8bit`，该 `PES Pack` 中该值为 `03`，即该 `PES Pack` 中，该字段之后 `3` 个字节为 `PES Header` 字段，剩余数据均为数据 `Payload`。

## PS Stream B

### PS Header Extern

如果 `PS Pack` 第 `10` 字节后三位不是 `000(2)`，则代表其后有相应长度的拓展内容，比如下图码流中就有 `6` 字节拓展内容：

![拓展内容 2](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117220016.png)

### Program Stream Map

如果码流中遇到了 `0x00 00 01 bc`，则之后的数据为 `Program Stream Map` 数据。

![Program Stream Map](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117223419.png)

`Program Stream Map`，即 `PSM`，其格式如下图所示：

![Program Stream Map 格式](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117223519.png)

可以看出，`0x00 00 01 bc` 后两位代表该部分数据长度，该 `PSM` 数据长度为 `0x62`，转化为十进制，即 `98` 字节。

![Program Stream Map length](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117223909.png)

结合上边的 `PSM` 结构，自 `0x00 62` 后，跳过两个字节的固定内容，就到了两个字节的 `program_stream_info_length`，其值为 `0x00 2c`，说明其后跟着的 `descriptor` 共占 `44` 字节。

![descriptor](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117230813.png)

跳过该长度信息和 `44` 字节后，接下来的值 `0x00 2c` 为 `element_stream_map_length`（基本流映射长度），也就是 `44` 字节，它表示接下来的 `44` 字节都是用来描述原始流信息的。

![c](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117231212.png)

接下来进入了原始流描述的第一次循环，第一个字节 `stream_type` 值为 `0x24`，根据《GB/T 28181-2016 修改补充文件》可知其为 `H.265` 码流。

        a)  MPEG-4  视频流：    0x10
        b)  H.264   视频流：    0x1B
        c)  SVAC    视频流：    0x80
        d)  H.265   视频流：    0x24
        e)  G.711A  音频流：    0x90
        f)  G.711U  音频流：    0x91
        g)  G.722.1 音频流：    0x92
        h)  G.723.1 音频流：    0x93
        i)  G.729   音频流：    0x99
        j)  SVAC    音频流：    0x9B

之后的 `0xe0` 表示其为视频流。

![stream_type](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117232429.png)

之后的 `0x00 10` 代表该视频流描述 `element_stream_info_length` 占 `16` 字节。

![element_stream_info_length](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117232540.png)

同理，接着循环下去，接下来的字段 `0x91` 和 `0xc0`，代表着其为 `G.711U` 编码的音频流，并携带有 `0x0c` 即 `12` 字节的描述信息。

![stream_type Audio](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117232848.png)

继续循环，接下来的字段 `0xbd bd`，是海康的私有数据，携带有 `0` 字节的描述信息。

继续循环，接下来的字段 `0xbf bf`，也是海康的私有数据，携带有 `0` 字节的描述信息。

![私有数据 PSM](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201117233803.png)

以上四次循环后，他们占用的字节数已经等于 `44`，跟 `element_stream_map_length` 值相等。

接下来的内容就是 `CRC_32`，`4bits`，从以下码流中可以看出，海康 IPC 中没有对这些字节进行计算。

![CRC_32](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201118211141.png)

### PTS_DTS_flags

在很多码流中，都会存在 `PTS` 或 `DTS` 字段，此时 `PTS_DTS_flags` 字段就会置位，比如以下码流中，`PTS_DTS_flags` 为 `00(2)`

![PTS_DTS_flags](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201118212717.png)

对比 {% post_link "音视频封装 - PES 封装格式" %} 中 `PES` 码流结构可知，之后由 `5` 字节的 `PTS` 数据

![PES Struct](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201118213841.png)

![PTS Data](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201118214049.png)

### ES Stream

经过以上步骤分析出 `PS` 的各层封装后，就能将封装中的 `ES` 码流提取出来：

![ES](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201118212109.png)

## 参考资料

- [1] 文中 `PS` 码流 `A` 视频源文件：[点击下载](/slave/PS/TestVideoA.7z "密码：github.com")
- [2] 文中 `PS` 码流 `B` 视频源文件：[点击下载](/slave/PS/TestVideoB.7z "密码：github.com")
- [3] {% post_link "音视频封装 - PS 封装格式" %}
- [4] {% post_link "音视频封装 - PES 封装格式" %}
