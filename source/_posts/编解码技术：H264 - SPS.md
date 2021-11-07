---
title: 编解码技术：H264 - SPS
categories:
  - 编解码技术
tags:
  - H.264
abbrlink: 4b4fd3b8
date: 2020-08-04 13:59:51
---
## SPS 简介

在 H.264 标准协议中规定了多种不同的 NAL Unit 类型，其中类型 7 表示该 NAL Unit 内保存的数据为 Sequence Paramater Set（序列参数集）。

在 H.264 的各种语法元素中，SPS 中的信息至关重要。如果其中的数据丢失或出现错误，那么解码过程很可能会失败。

SPS 即 Sequence Paramater Set，又称作序列参数集。SPS 中保存了一组编码视频序列 (Coded
videosequence) 的全局参数。所谓的编码视频序列即原始视频的一帧一帧的像素数据经过编码之后的结构组成的序列。而每一帧的编码后数据所依赖的参数保存于图像参数集中。一般情况 SPS 和 PPS 的 NAL Unit 通常位于整个码流的起始位置。但在某些特殊情况下，在码流中间也可能出现这两种结构，主要原因可能为：

- 解码器需要在码流中间开始解码
- 编码器在编码的过程中改变了码流的参数（如图像分辨率等）

<!-- more -->

## SPS 语法元素及其含义

在编解码过程中，如果需要使用 SPS 中包含的参数，必须对其中的数据进行解析。其中 H.264 标准协议中规定的 SPS 格式位于文档的 7.3.2.1 部分，如下图所示：

![Sequence parameter set data syntax](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200804220842.jpg)

其中的每一个语法元素及其含义如下：

### profile_idc

标识当前 H.264 码流的 profile，H.264 中定义了三种常用的档次 profile：

- 基准档次：baseline profile;

- 主要档次：main profile;

- 扩展档次：extended profile;

> 参见：{% post_link "编解码技术：H264 - Profile" %}

#### 可选参数

根据上图所示，在 profile_idc 取值为一些特殊值时，存在一些可选参数

##### chroma_format_idc

- chroma_format_idc 是指与亮度取样对应的色度取样。chroma_format_idc 的值应该在 0
到 3 的范围内（包括 0 和 3）。
- chroma_format_idc 不存在时，应推断其值为 1（4：2：0 的色度格式）。

![SubWidthC, and SubHeightC values derived from chroma_format_idc and separate_colour_plane_flag](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200808192638.jpg)

##### separate_colour_plane_flag

- separate_colour_plane_flag 等于 1 表示三个颜色分量 Y、Cb、Cr 分别进行编码。

- split_colour_plane_flag 等于 0 表示颜色分量未单独编码。

根据 separate_colour_plane_flag 的值，变量 ChromaArrayType 的值有如下规定：

- 如果 eparate_colour_plane_flag 等于 0，则将 ChromaArrayType 设置为等于 chroma_format_idc。
- 否则（separate_colour_plane_flag 等于 1），将 ChromaArrayType 设置为等于 0。

##### bit_depth_luma_minus8

bit_depth_luma_minus8 是指亮度队列样值的比特深度以及亮度量化参数范围的取值偏移 QpBdOffsetY ，如
下所示：

    BitDepthY = 8 + bit_depth_luma_minus8
    QpBdOffsetY = 6 * bit_depth_luma_minus8

- 当 bit_depth_luma_minus8 不存在时，应推定其值为 0。
- bit_depth_luma_minus8 取值范围应该在 0 到 4 之间（包括 0 和 4）。

##### bit_depth_chroma_minus8

bit_depth_chroma_minus8 是指色度队列样值的比特深度以及色度量化参数范围的取值偏移 QpBdOffsetC，如下所示：

    BitDepthC = 8 + bit_depth_chroma_minus8
    QpBdOffsetC = 6 * (bit_depth_chroma_minus8 + residual_colour_transform_flag)

- 当 bit_depth_chroma_minus8 不存在时，应推定其值为 0。
- bit_depth_chroma_minus8 取值范围应该在 0 到 4 之间（包括 0 和 4）。

> 当 ChromaArrayType 等于 0 时，在解码过程中不使用 bit_depth_chroma_minus8 的值。特别是，当 separate_colour_plane_flag 等于 1 时，使用亮度分量解码过程将每个颜色平面解码为一个不同的单色图片（除了 比例矩阵的选择）和亮度位深度用于所有三个颜色分量。

变量 RawMbBits 按下列公式得出：

    RawMbBits = 256 * BitDepthY + 2 * MbWidthC * MbHeightC * BitDepthC

##### qpprime_y_zero_transform_bypass_flag

- qpprime_y_zero_transform_bypass_flag 等于 1 是指当 QP'Y 等于 0 时变换系数解码过程的变换旁路操作和图像构建过程将会在 H264 官方文档第 8.5 节给出的去块效应滤波过程之前执行。
- qpprime_y_zero_transform_bypass_flag 等于 0 是指变换系数解码过程和图像构建过程在去块效应滤波过程之前执行而不使用变换旁路操作。
- 当 qpprime_y_zero_transform_bypass_flag 没有特别指定时，应推定其值为 0。

##### seq_scaling_matrix_present_flag

- seq_scaling_matrix_present_flag 等于 1 表示存在 i = 0..7 的标志 seq_scaling_list_present_flag[i] 。
- seq_scaling_matrix_present_flag 等于 0 则表示不存在这些标志并且由 Flat_4x4_16 表示的序列级别的缩放比例列表应被推断出来（对应 i = 0..5），由 Flat_8x8_16 表示的序列级别的缩放比例列表应被推断出来（对应 i = 6..7）。
- 当 seq_scaling_matrix_present_flag 没有特别指定时，应推定其值为 0。

缩放比例列表 Flat_4x4_16 和 Flat_8x8_16 规定如下：
    Flat_4x4_16[i] = 16, i = 0..15,
    Flat_8x8_16[i] = 16, i = 0..63.

seq_scaling_list_present_flag[i] 等于 1 是指视频序列参数集中存在缩放比例列表 i 的语法结构。
seq_scaling_list_present_flag[i] 等于 0 表示视频序列参数集中不存在缩放比例列表 i 的语法结构并且 H264 官方文档表 7-2 中列出的缩放比例序列后退规则集 A 应用于以 i 值为索引的序列级别的缩放比例列表。

![缩放比例列表的记忆名索引号分配以及后退规则的规范](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200808194737.png)
![默认缩放比例列表 Default_4x4_Intra 和 Default_4x4_Inter 的规范](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200808194822.png)

### constraint_set*_flag

constraint_set0_flag ~ constraint_set5_flag 是在编码的档次方面对码流增加的其他一些额外限制性条件：

#### constraint_set0_flag

- constraint_set0_flag 等于 1 表示编码视频序列遵守 Baseline profile 的所有约束。

- constraint_set0_flag 等于 0 表示编码视频序列可能遵守也可能不遵守条款 Baseline profile 的所有约束。

#### constraint_set1_flag

- constraint_set1_flag 等于 1 表示编码视频序列遵守 Main profile 的所有约束。

- constraint_set1_flag 等于 0 表示编码视频序列可能遵守也可能不遵守 Main profile 的所有约束。

#### constraint_set2_flag

- constraint_set2_flag 等于 1 表示编码视频序列遵守 Extended profile 的所有约束。

- constraint_set2_flag 等于 0 表示编码视频序列可能遵守也可能不遵守 Extended profile 的所有约束。

> 当 constraint_set0_flag，constraint_set1_flag 或 constraint_set2_flag 中的一个或多个等于 1 时，编码的视频序列必须服从官方文档 附录 A.2 中所有指示的子句的约束。
> 当 profile_idc 等于 44、100、110、122 或 244 时，constraint_set0_flag，constraint_set1_flag 和 constraint_set2_flag 的值都必须等于 0。

#### constraint_set3_flag

- 如果 profile_idc 等于 66、77 或 88 并且 level_ idc 等于 11，constraint_set3_flag 等于 1 是指该比特流遵从 H264 官方文档附件 A 中对 Level 1.1 的所有规定；constraint_set3_flag 等于 0 是指该比特流可以遵从也可以不遵从附件 A 中有关 1b 级别的所有规定。
- 否则，如果 profile_idc 等于 100 或 110，constraint_set3_flag 等于 1 表示编码视频
序列遵循 H264 官方文档附件 A 中针对 High 10 Intra Profile 指定的所有约束，constraint_set3_flag 等于 0 表示编码视频序列可能遵守也可能不遵守这些相应的约束。
- 否则，如果 profile_idc 等于 122，constraint_set3_flag 等于 1 表示编码视频序列服从
 H264 官方文档附件 A 中针对高 4：2：2 帧内配置文件指定的所有约束，constraint_set3_flag 等于 0 表示编码的视频序列可能会也可能不会遵守这些相应的约束。
- 否则，如果 profile_idc 等于 44，constraint_set3_flag 应等于 1。当 profile_idc 等于 44 时，不允许 constraint_set3_flag 的值等于 0。
- 否则，如果 profile_idc 等于 244，constraint_set3_flag 等于 1 表示编码视频序列服从
 H264 官方文档附件 A 中针对高 4：4：4 帧内配置文件指定的所有约束，且 constraint_set3_flag 等于 0 表示编码的视频序列可能会也可能不会遵守这些相应的约束。
- 否则（profile_idc 等于 66、77 或 88，level_idc 不等于 11，或者 profile_idc 不等于 66、77、88、100、110、122、244 或 44），constraint_set3_flag 的值 1 保留给 ITU-T | ISO / IEC。 对于 profile_idc 等于 66、77 或 88，level_idc 不等于 11 的编码视频序列，以及 profile_idc 不等于 66、77、88、100、110、122、244 或 44 的编码视频序列，constraint_set3_flag 应当等于 0，符合本建议书的比特流 | 国际标准。 当 profile_idc 等于 66、77 或 88 且 level_idc 不等于 11 或 profile_idc 不等于 66、77、88、100、110、122、244 或 44 时，解码器应忽略 constraint_set3_flag 的值。

> constraint_set4_flag ~ constraint_set5_flag 详见 H.264 官方文档

### level_idc

标识当前码流的 Level。编码的 Level 定义了某种条件下的最大视频分辨率、最大视频帧率等参数，码流所遵从的 level 由 level_idc 指定。

> 参见：{% post_link "编解码技术：H264 - Level" %}

### 其他参数

#### seq_parameter_set_id

表示当前的序列参数集的 id。通过该 id 值，图像参数集 pps 可以引用其代表的 sps 中的参数。

#### log2_max_frame_num_minus4

这个句法元素主要是为读取另一个句法元素 frame_num 服务的，frame_num 是最重要的句法元素之一，它标识所属图像的解码顺序。可以在句法表看到，frame_num 的解码函数是 ue（v），函数中的 v 在这里指定：

    v = log2_max_frame_num_minus4 + 4

从另一个角度看，这个句法元素同时也指明了 frame_num 的所能达到的最大值 MaxFrameNum，计算公式为：

    MaxFrameNum = 2^(log2_max_frame_num_minus4 + 4)

值得注意的是 frame_num 是循环计数的，即当它到达 MaxFrameNum 后又从 0 重新开始新一轮
的计数。解码器必须要有机制检测这种循环，不然会引起类似千年虫的问题，在图像的顺序上造成
混乱。

#### pic_order_cnt_type

表示解码 picture order count(POC) 的方法。POC 是另一种计量图像序号的方式，与 frame_num 有着不同的计算方法。POC 标识图像的播放顺序。由于 H.264 使用了 B 帧预测，使得图像的解码顺序并不一定等于播放顺序，但它们之间存在一定的映射关系。POC 可以由 frame-num 通过映射关系计算得来，也可以索性由编码器显式地传送。H.264 中一共定义了三种 POC 的编码方法，这个句法元素就是用来通知解码器该用哪种方法来计算 POC。而以下的几个句法元素是分别在各种方法中用到的数据。

在如下的视频序列中本句法元素不应该等于 2：

- 一个非参考帧的接入单元后面紧跟着一个非参考图像（指参考帧或参考场）的接入单元
- 两个分别包含互补非参考场对的接入单元后面紧跟着一个非参考图像的接入单元.
- 一个非参考场的接入单元后面紧跟着另外一个非参考场, 并且这两个场不能构成一个互补场对

本句法元素的取值为 0、1 或 2。

#### log2_max_pic_order_cnt_lsb_minus4

用于计算 MaxPicOrderCntLsb 的值，该值表示 POC 的上限。计算公式为：

    MaxPicOrderCntLsb = 2^(log2_max_pic_order_cnt_lsb_minus4 + 4)。

本句法元素在 pic_order_cnt_type = 0 时使用。

#### delta_pic_order_always_zero_flag

本句法元素等于 1 时, 句法元素 delta_pic_order_cnt[0] 和 delta_pic_order_cnt[1] 不在片头出现, 并且它们的值默认为 0;

本句法元素等于 0 时, 上述的两个句法元素将在片头出现。

#### offset_for_non_ref_pic

本句法元素被用来计算非参考帧或场的 picture order count(POC), 本句法元素的值应该在 `[-231 , 231 – 1]` 范围内。

#### offset_for_top_to_bottom_field

本句法元素被用来计算帧的底场的 picture order count(POC), 本句法元素的值应该在 `[-231 , 231 – 1]` 范围内。

#### num_ref_frames_in_pic_order_cnt_cycle

本句法元素被用来解码 picture order count(POC), 本句法元素的值应该在 [0,255] 范围内。

#### offset_for_ref__frame

offset_for_ref__frame[i] 在 picture order count type=1 时用，用于解码 POC，本句法元素对循环 num_ref_frames_in_pic_order_cycle 中的每一个元素指定一个偏移。

#### max_num_ref_frames

用于表示参考帧的最大数目。

解码器可依照这个句法元素的值开辟存储区，这个存储区用于存放已解码的参考帧，H.264 规定最多可用 16 个参考帧，本句法元素的值最大为 16。值得注意的是这个长度以帧为单位，如果在场模式下，应该相应地扩展一倍。

#### gaps_in_frame_num_value_allowed_flag

标识位，说明 frame_num 中是否允许不连续的值。

这个句法元素等于 1 时，表示允许句法元素 frame_num 可以不连续。当传输信道堵塞严重时，编码器来不及将编码后的图像全部发出，这时允许丢弃若干帧图像。在正常情况下每一帧图像都有依次连续的 frame_num 值，解码器检查到如果 frame_num 不连续，便能确定有图像被编码器丢弃。这时，解码器必须启动错误掩藏的机制来近似地恢复这些图像，因为这些图像有可能被后续图像用作参考帧。

当这个句法元素等于 0 时，表不允许 frame_num 不连续，即编码器在任何情况下都不能丢弃图像。
这时，H.264 允许解码器可以不去检查 frame_num 的连续性以减少计算量。这种情况下如果依然发
生 frame_num 不连续，表示在传输中发生丢包，解码器会通过其他机制检测到丢包的发生，然后启
动错误掩藏的恢复图像。

#### pic_width_in_mbs_minus1

用于计算图像的宽度。单位为宏块个数：

    PicWidthInMbs = pic_width_in_mbs_minus1 + 1     /* 以宏块为单位的图像宽度 */

通过这个句法元素解码器可以计算得到亮度和色度分量以像素为单位的图像宽度：

    PicWidthInSamplesL = PicWidthInMbs * 16         /* 亮度分量图像宽度（像素） */
    PicWidthInSamplesC = PicWidthInMbs * MbWidthC   /* 色度分量图像宽度（像素） */

H.264 将图像的大小在序列参数集中定义，意味着可以在通信过程中随着序列参数集动态地改变图
像的大小，甚至可以将传送的图像剪裁后输出。

#### pic_height_in_map_units_minus1

使用 PicHeightInMapUnits 来度量视频中一帧图像的高度。PicHeightInMapUnits 并非图像明确的以像素或宏块为单位的高度，而需要考虑该宏块是帧编码或场编码。PicHeightInMapUnits 和 PicSizeInMapUnits 的计算方式为：

    PicHeightInMapUnits = pic_height_in_map_units_minus1 + 1
    PicSizeInMapUnits = PicWidthInMbs * PicHeightInMapUnits

注意：PicHeightInMapUnits 和 PicSizeInMapUnits 是以 map_unit 为单位。

#### frame_mbs_only_flag

标识位，说明宏块的编码方式。

本句法元素等于 1 时表示本序列中所有图像的编码模式都是帧，没有其他编码模式存在；

本句法元素等于 0 时表示本序列中图像的编码模式可能是帧，也可能是场或帧场自适应，某个图像具体是哪一种要由其他句法元素决定。

根据该标识位取值不同，PicHeightInMapUnits 的含义也不同，为 0 时表示一场数据按宏块计算的高度，为 1 时表示一帧数据按宏块计算的高度。

结合 map_unit 的含义，这里给出上一个句法元素 pic_height_in_map_units_minus1 的进一步解析步骤：

- 当 frame_mbs_only_flag 等于 1，pic_height_in_map_units_minus1 指的是一个 picture 中帧的高度；
- 当 frame_mbs_only_flag 等于 0，pic_heght_in_map_units_minus1 指的是一个 picture 中场的高度，所以可以得到如下以宏块为单位的图像高度：

    FrameHeightInMbs = (2 – frame_mbs_only_flag) * PicHeightInMapUnits

#### mb_adaptive_frame_field_flag

标识位，说明是否采用了宏块级的帧场自适应编码。当该标识位为 0 时，不存在帧编码和场编码之间的切换；当标识位为 1 时，宏块可能在帧编码和场编码模式之间进行选择。

指明本序列是否属于帧场自适应模式。

- mb_adaptive_frame_field_flag 等于 1 时表明在本序列中的图像如果不是场模式就是帧场自适应模式，

- mb_adaptive_frame_field_flag 等于 0 时表示本序列中的图像如果不是场模式就是帧模式。

以下列举了一个序列中可能出现的编码模式：

1. 全部是帧，对应于 frame_mbs_only_flag = 1 的情况。
2. 帧和场共存。frame_mbs_only_flag = 0, mb_adaptive_frame_field_flag = 0
3. 帧场自适应和场共存。frame_mbs_only_flag = 0, mb_adaptive_frame_field_flag = 1

注意：帧和帧场自适应不能共存在一个序列中。

#### direct_8x8_inference_flag

标识位，用于 B_Skip、B_Direct 模式运动矢量的推导计算。

#### frame_cropping_flag

标识位，说明是否需要对输出的图像帧进行裁剪，如果是的话，后面紧跟着的四个句法元素分别指出左右、上下裁剪的宽度。

#### frame_crop_xxx_offset

如上一句法元素所述，包括：

- frame_crop_left_offset
- frame_crop_right_offset
- frame_crop_bottom_offset
- frame_crop_bottom_offset

#### vui_parameters_present_flag

标识位，说明 SPS 中是否存在 VUI 信息。VUI 用以表征视频格式等额外信息。

## SPS 示例

    0000   67 4d 00 32 96 35 40 50 01 6b 4d c0 40 40 50 00   gM.2.5@P.kM.@@P.
    0010   00 70 80 00 15 f9 00 40                           .p.....@

    ============================= Wirshark 解析 =============================
    H.264
        NAL unit header or first byte of the payload
            0... .... = F bit: No bit errors or other syntax violations
            .11. .... = Nal_ref_idc (NRI): 3
            ...0 0111 = Type: NAL unit - Sequence parameter set (7)
        H264 NAL Unit Payload
            0100 1101 = Profile_idc: Main profile (77)
            0... .... = Constraint_set0_flag: 0
            .0.. .... = Constraint_set1_flag: 0
            ..0. .... = Constraint_set2_flag: 0
            ...0 .... = Constraint_set3_flag: 0
            .... 0... = Constraint_set4_flag: 0
            .... .0.. = Constraint_set5_flag: 0
            .... ..00 = Reserved_zero_2bits: 0
            0011 0010 = Level_id: 50 [Level 5.0 135 Mb/s]
            1... .... = seq_parameter_set_id: 0
            .001 01.. = log2_max_frame_num_minus4: 4
            .... ..1. = pic_order_cnt_type: 0
            .... ...0  0011 01.. = log2_max_pic_order_cnt_lsb_minus4: 12
            .... ..01  0... .... = num_ref_frames: 1
            .1.. .... = gaps_in_frame_num_value_allowed_flag: 1
            ..00 0000  0101 0000  0... .... = pic_width_in_mbs_minus1: 159
            .000 0001  0110 10.. = pic_height_in_map_units_minus1: 89
            .... ..1. = frame_mbs_only_flag: 1
            .... ...1 = direct_8x8_inference_flag: 1
            0... .... = frame_cropping_flag: 0
            .1.. .... = vui_parameters_present_flag: 1
            ..0. .... = aspect_ratio_info_present_flag: 0
            ...0 .... = overscan_info_present_flag: 0
            .... 1... = video_signal_type_present_flag: 1
            .... .101 = video_format: Unspecified video format (5)
            1... .... = video_full_range_flag: 1
            .1.. .... = colour_description_present_flag: 1
            ..00 0000  01.. .... = colour_primaries: 1
            ..00 0000  01.. .... = transfer_characteristics: 1
            ..00 0000  01.. .... = matrix_coefficients: 1
            ..0. .... = chroma_loc_info_present_flag: 0
            ...1 .... = timing_info_present_flag: 1
            .... 0000  0000 0000  0000 0000  0111 0000  1000 .... = num_units_in_tick: 1800
            .... 0000  0000 0000  0001 0101  1111 1001  0000 .... = time_scale: 90000
            .... 0... = fixed_frame_rate_flag: 0
            .... .0.. = nal_hrd_parameters_present_flag: 0
            .... ..0. = vcl_hrd_parameters_present_flag: 0
            .... ...0 = pic_struct_present_flag: 0
            0... .... = bitstream_restriction_flag: 0
            .1.. .... = rbsp_stop_bit: 1
            ..00 0000 = rbsp_trailing_bits: 0

## 参考资料

* [1] 《 T-REC-H.264-201906-I.pdf 》
* [2] [H264 码流中 SPS PPS 详解 - 知乎](https://zhuanlan.zhihu.com/p/27896239)
* [3] 《 新一代视频压缩编码标准 -- H264/AVC（第二版）》
