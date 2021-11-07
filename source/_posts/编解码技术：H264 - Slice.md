---
title: 编解码技术：H264 - Slice
categories:
  - 编解码技术
tags:
  - H.264
abbrlink: '92167482'
date: 2020-08-10 15:24:02
---
## Slice

一个 Slice 包含一帧图像的部分或全部数据，换言之，一帧视频图像可以编码为一个或若干个 Slice。一个 Slice 最少包含一个宏块，最多包含整帧图像的数据。在不同的编码实现中，同一帧图像中所构成的 Slice 数目不一定相同。

在 H.264 中设计 Slice 的目的主要在于防止误码的扩散。因为不同的 slice 之间，其解码操作是独立的。某一个 slice 的解码过程所参考的数据（例如预测编码）不能越过 slice 的边界。

<!-- more -->

### Slice 的类型

根据码流中不同的数据类型，H.264 标准中共定义了 5 种 Slice 类型：

- I slice: 帧内编码的条带；
- P slice: 单向帧间编码的条带；
- B slice: 双向帧间编码的条带；
- SI slice: 切换 I 条带，用于扩展档次中码流切换使用；
- SP slice: 切换 P 条带，用于扩展档次中码流切换使用；

在 I slice 中只包含 I 宏块，不能包含 P 或 B 宏块；在 P 和 B slice 中，除了相应的 P 和 B 类型宏块之外，还可以包含 I 类型宏块。

### Slice 的组成

每一个 Slice 总体来看都由两部分组成，一部分作为 Slice header，用于保存 Slice 的总体信息（如当前 Slice 的类型等），另一部分为 Slice body，通常是一组连续的宏块结构（或者宏块跳过信息），如下图所示：

![Slice 的组成](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200811225155.png)

### Slice Header 结构

Slice header 中主要保存了当前 slice 的一些全局的信息，slice body 中的宏块在进行解码时需依赖这些信息。Slice Header 结构如下图所示：

![Slice Header 结构](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200811230024.jpg)

### Slice Header 语义

* first_mb_in_slice 表示在条带中第一个宏块的地址。

    当如附件 A 中规定的那样不允许任意的条带顺序时，本条带的 first_mb_in_slice 的值应不小于当前图像的任何在该条带之前（按解码顺序）的其他条带的 first_mb_in_slice 的值。
    条带中第一个宏块的地址按如下方式得到：
    - 如果 MbaffFrameFlag 等于 0， first_mb_in_slice 就是该条带中第一个宏块的地址，并且 first_mb_in_slice 的值应在 0 到 PicSizeInMbs – 1 的范围内（包括边界值）。
    - 否则（MbaffFrameFlag 等于 1），first_mb_in_slice * 2 就是该条带中的第一个宏块地址，该宏块是该条带中第一个宏块对中的顶宏块，并且 first_mb_in_slice 的值应该在 0 到 PicSizeInMbs / 2 – 1 的范围内（包括边界值）

* slice_type 表示条带的编码类型：

    ![Name association to slice_type](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200811230957.jpg)

    slice_type 的值在 5 到 9 范围内表示，除了当前条带的编码类型，所有当前编码图像的其他条带的 slice_type 值应与当前条带的 slice_type 值 一样，或者等于当前条带的 slice_type 值减 5。
    当 nal_unit_type 等于 5（IDR 图片）时，slice_type 应等于 2、4、7 或 9。
    当 max_num_ref_frames 等于 0 时，slice_type 应等于 2、4、7 或 9。

* pic_parameter_set_id 指定使用的图像参数集。

    pic_parameter_set_id 的值应该在 0 到 255 范围内（包括 0 和 255）。

* frame_num 用作一个图像标识符。

    frame_num 在比特流中应由 log2_max_frame_num_minus4 + 4 个比特表示。关于 frame_num 的规定如下：
    变量 PrevRefFrameNum 按如下方式得到：
    - 如果当前的图像是一个 IDR 图像， PrevRefFrameNum 将设为 0。
    - 否则（当前图像不是一个 IDR 图像），PrevRefFrameNum 设置如下：
        - 如果对包含一个非参考图像，且按解码顺序其前面一个访问单元包含一个参考图像的访问单元解码的过程调用 8.2.5.2 节规定的 frame_num 出现中断的解码过程，PrevRefFrameNum 应设为等于最后一个 “不存在的”，由 8.2.5.2 节规定的 frame_num 出现中断的解码过程推定的参考帧的 frame_num 值。
        - 否则 PrevRefFrameNum 设为等于按解码过程包含参考图像的前一个访问单元的 frame_num 值。

    frame_num 的取值规定如下：
    - 如果当前图像是一个 IDR 图像， frame_num 应等于 0。
    - 否则（当前图像不是一个 IDR 图像），则按解码顺序前一个访问单元包含一个参考图像作为前导参考图像，该访问单元中有基本编码图像，当前图像的 frame_num 值不应等于 PrevRefFrameNum ，除非满足下面所有三个条件：
        * 按解码顺序当前图像和前导参考图像属于连续的访问单元。
        * 当前图像和前导参考图像是具有相反奇偶性的参考场。
        * 满足下面的一个或多个条件：
            - 前导参考图像是一个 IDR 图像
            - 前导参考图像包括一个值等于 5 的 memory_management_control_语法元素
                * 注：当前导参考图像包括一个 memory_management_control_operation 语法元素等于 5 时，PrevRefFrameNum 应等于 0。
            - 在前导参考图像之前存在一个基本编码图像，并且该基本编码图像的 frame_num 不等于 PrevRefFrameNum。
            - 在前导参考图像之前存在一个基本编码图像，并且该基本编码图像不是一个参考图像。

    当 frame_num 的值不等于 PrevRefFrameNum 时，下列几条将被应用：
    - 按解码顺序，不应有任何前面的场或帧当前被标记为 “用于短期参考” 并具有一个 frame_num 值等于由变量 UnusedShortTermFrameNum 按下列公式得出的任何值：
        > UnusedShortTermFrameNum = (PrevRefFrameNum + 1) % MaxFrameNum while( UnusedShortTermFrameNum != frame_num )
        > UnusedShortTermFrameNum = (UnusedShortTermFrameNum + 1) % MaxFrameNum
    - frame_num 的值规定如下：
        * 如果 gaps_in_frame_num_value_allowed_flag 等于 0 ， 当前图像的 frame_num 值将等于 (PrevRefFrameNum + 1) % MaxFrameNum。
        * 否则（gaps_in_frame_num_value_allowed_flag 等于 1），下面几条将应用：
            - 如果 frame_num 大于 PrevRefFrameNum，按解码顺序比特流中在前面参考图像之后应不会有任何非参考图像在当前图像之前，且此段解码序列满足下面两个条件。
                * 非参考图像的 frame_num 值小于 PrevRefFrameNum。
                * 非参考图像的 frame_num 值大于当前图像的 frame_num 值。
            - 否则（frame_num 小于 PrevRefFrameNum），按解码顺序比特流中在前面参考图像之后应不会有任何非参考图像在当前图像之前，且此段解码序列满足下面两个条件。
                * 非参考图像的 frame_num 的值小于 PrevRefFrameNum。
                * 非参考图像的 frame_num 值大于当前图像的 frame_num 值。

## Slice Group
