---
title: 编解码技术：H264 - PPS
categories:
  - 编解码技术
tags:
  - H.264
abbrlink: 49096de1
date: 2020-08-14 15:48:47
---
## PPS 简介

除了序列参数集 SPS 之外，H.264 中另一重要的参数集合为图像参数集 Picture Paramater Set(PPS)。通常情况下，PPS 类似于 SPS，在 H.264 的裸码流中单独保存在一个 NAL Unit 中，只是 PPS NAL Unit 的 nal_unit_type 值为 8；而在封装格式中，PPS 通常与 SPS 一起，保存在视频文件的文件头中。

## PPS 语法元素及其含义

在编解码过程中，如果需要使用 PPS 中包含的参数，必须对其中的数据进行解析。其中 H.264 标准协议中规定的 PPS 格式位于文档的 7.3.2.2 部分，如下图所示：

<!-- more -->

![Picture parameter set RBSP syntax](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200815224220.jpg)

其中的每一个语法元素及其含义如下：

### 必选参数部分

#### pic_parameter_set_id

pic_parameter_set_id 表示当前 PPS 的 id。某个 PPS 在码流中会被相应的 slice 引用，slice 引用 PPS 的方式就是在 Slice header 中保存 PPS 的 id 值。该值的取值范围为 [0,255]。

#### seq_parameter_set_id

seq_parameter_set_id 是指活动的序列参数集。变量 seq_parameter_set_id 的值应该在 0 到 31 的范围内（包括 0 和 31）。

#### entropy_coding_mode_flag

entropy_coding_mode_flag 用于选取语法元素的熵编码方式，在语法表中由两个标识符代表，具体如下：

- 如果 entropy_coding_mode_flag 等于 0，那么采用语法表中左边的描述符所指定的方法
- 否则（entropy_coding_mode_flag 等于 1），就采用语法表中右边的描述符所指定的方法

例如：在一个宏块语法元素中，宏块类型 mb_type 的语法元素描述符为 “ue(v) | ae(v)”，在 baseline profile 等设置下采用指数哥伦布编码，在 main profile 等设置下采用 CABAC 编码。

#### bottom_field_pic_order_in_frame_present_flag

标识位，用于表示另外条带头中的两个语法元素 delta_pic_order_cnt_bottom 和 delta_pic_order_cn 是否存在的标识。这两个语法元素表示了某一帧的底场的 POC 的计算方法。

#### num_slice_groups_minus1

num_slice_groups_minus1 加 1 表示一个图像中的条带组数。当 num_slice_groups_minus1 等于 0 时，图像中所有的条带属于同一个条带组。

#### num_ref_idx_l0_active_minus1

num_ref_idx_l0_active_minus1 表示参考图像列表 0 的最大参考索引号，该索引号将用来在一幅图像中 num_ref_idx_active_override_flag 等于 0 的条带使用列表 0 预测时，解码该图像的这些条带。当 MbaffFrameFlag 等于 1 时，num_ref_idx_l0_active_minus1 是帧宏块解码的最大索引号值，而 2 * num_ref_idx_l0_active_minus1 + 1 是场宏块解码的最大索引号值。num_ref_idx_l0_active_minus1 的值应该在 0 到 31 的范围内（包括 0 和 31）。

#### num_ref_idx_l1_active_minus1

num_ref_idx_l1_active_minus1 与 num_ref_idx_l0_active_minus1 具有同样的定义，只是分别用 11 和列表 1 取代 10 和列表 0。

#### weighted_pred_flag

weighted_pred_flag 等于 0 表示加权的预测不应用于 P 和 SP 条带。weighted_pred_flag 等于 1 表示在 P 和 SP 条带中应使用加权的预测。

#### weighted_bipred_idc

weighted_bipred_idc 的值应该在 0 到 2 之间（包括 0 和 2）:

- weighted_bipred_idc 等于 0 表示 B 条带应该采用默认的加权预测。
- weighted_bipred_idc 等于 1 表示 B 条带应该采用具体指明的加权预测。
- weighted_bipred_idc 等于 2 表示 B 条带应该采用隐含的加权预测。

#### pic_init_qp_minus26

pic_init_qp_minus26 表示每个条带的 SliceQPY 初始值减 26。当解码非 0 值的 slice_qp_delta 时，该初始值在条带层被修正，并且在宏块层解码非 0 值的 mb_qp_delta 时进一步被修正。pic_init_qp_minus26 的值应该在 -(26 + QpBdOffsetY) 到 +25 之间（包括边界值）。

#### pic_init_qs_minus26

pic_init_qs_minus26 表示在 SP 或 SI 条带中的所有宏块的 SliceQSY 初始值减 26。当解码非 0 值的 slice_qs_delta 时，该初始值在条带层被修正。pic_init_qs_minus26 的值应该在 -26 到 +25 之间（包括边界值）。

#### chroma_qp_index_offset

chroma_qp_index_offset 表示为在 QPC 值的表格中寻找 Cb 色度分量而应加到参数 QPY 和 QSY 上的偏移。chroma_qp_index_offset 的值应在 -12 到 +12 范围内（包括边界值）。

#### deblocking_filter_control_present_flag

- deblocking_filter_control_present_flag 等于 1 表示控制去块效应滤波器的特征的一组语法元素将出现在条带头中。
- deblocking_filter_control_present_flag 等于 0 表示控制去块效应滤波器的特征的一组语法元素不会出现在条带头中，并且它们的推定值将会生效。

#### constrained_intra_pred_flag

- constrained_intra_pred_flag 等于 0 表示帧内预测允许使用残余数据，且使用帧内宏块预测模式编码的宏块的预测可以使用帧间宏块预测模式编码的相邻宏块的解码样值。
- constrained_intra_pred_flag 等于 1 表示受限制的帧内预测，在这种情况下，使用帧内宏块预测模式编码的宏块的预测仅使用残余数据和来自 I 或 SI 宏块类型的解码样值。

#### redundant_pic_cnt_present_flag

- redundant_pic_cnt_present_flag 等于 0 表示 redundant_pic_cnt 语法元素不会在条带头、图像参数集中指明（直接或与相应的数据分割块 A 关联）的数据分割块 B 和数据分割块 C 中出现。
- redundant_pic_cnt_present_flag 等于 1 表示 redundant_pic_cnt 语法元素将出现在条带头、图像参数集中指明（直接或与相应的数据分割块 A 关联）的数据分割块 B 和数据分割块 C 中。

### 可选参数部分

#### slice_group_map_type

slice_group_map_type 表示条带组中条带组映射单元的映射是如何编码的。lice_group_map_type 的取值范围应该在 0 到 6 内（包括 0 和 6）。

- slice_group_map_type 等于 0 表示隔行扫描的条带组。
- slice_group_map_type 等于 1 表示一种分散的条带组映射。
- slice_group_map_type 等于 2 表示一个或多个前景条带组和一个残余条带组。
- slice_group_map_type 的值等于 3、4 和 5 表示变换的条带组。当 num_slice_groups_minus1 不等于 1 时，slice_group_map_type 不应等于 3、4 或 5。
- slice_group_map_type 等于 6 表示对每个条带组映射单元清楚地分配一个条带组。条带组映射单元规定如下：
    * 如果 frame_mbs_only_flag 等于 0，mb_adaptive_frame_field_flag 等于 1，而且编码图像是一个帧，那么条带组映射单元就是宏块对单元。
    * 否则，如果 frame_mbs_only_flag 等于 1 或者一个编码图像是一个场，条带组映射单元就是宏块的单元。
    * 否则（frame_mbs_only_flag 等于 0， mb_adaptive_frame_field_flag 等于 0，并且编码图像是一个帧），条带组映射单元就是像在一个 MBAFF 帧中的一个帧宏块对中一样垂直相邻的两个宏块的单元。

#### run_length_minus1

run_length_minus1[i] 用来指定条带组映射单元的光栅扫描顺序中分配给第 i 个条带组的连续条带组映射单元的数目。run_length_minus1[i] 的取值范围应该在 0 到 PicSizeInMapUnits – 1 内（包括边界值）。

#### top_left, bottom_right

top_left[i] 和 bottom_right[i] 分别表示一个矩形的左上角和右下角。top_left[i] 和 bottom_right[i] 是条带组映射单元所在图像的光栅扫描中的条带组映射单元位置。对于每个矩形 i，语法元素 top_left[i] 和 bottom_right[i] 的值都应该遵从下面所有的规定。

- `top_left[i] <= bottom_right[i] && bottom_right[i] < PicSizeInMapUnits`
- `(top_left[i] % PicWidthInMbs) < (bottom_right[i] % PicWidthInMbs)`

## PPS 示例

    0000   68 ee 3c 80                                       h.<.

    ============================= Wirshark 解析 =============================
    H.264
        NAL unit header or first byte of the payload
            0... .... = F bit: No bit errors or other syntax violations
            .11. .... = Nal_ref_idc (NRI): 3
            ...0 1000 = Type: NAL unit - Picture parameter set (8)
        H264 NAL Unit Payload
            1... .... = pic_parameter_set_id: 0
            .1.. .... = seq_parameter_set_id: 0
            ..1. .... = entropy_coding_mode_flag: 1
            ...0 .... = pic_order_present_flag: 0
            .... 1... = num_slice_groups_minus1: 0
            .... .1.. = num_ref_idx_l0_active_minus1: 0
            .... ..1. = num_ref_idx_l1_active_minus1: 0
            .... ...0 = weighted_pred_flag: 0
            00.. .... = weighted_bipred_idc: 0
            ..1. .... = pic_init_qp_minus26: 0
            ...1 .... = pic_init_qs_minus26: 0
            .... 1... = chroma_qp_index_offset: 0
            .... .1.. = deblocking_filter_control_present_flag: 1
            .... ..0. = constrained_intra_pred_flag: 0
            .... ...0 = redundant_pic_cnt_present_flag: 0
            1... .... = rbsp_stop_bit: 1
            .000 0000 = rbsp_trailing_bits: 0

## 参考资料

* [1] 《 T-REC-H.264-201906-I.pdf 》
* [2] [H264 码流中 SPS PPS 详解 - 知乎](https://zhuanlan.zhihu.com/p/27896239)
* [3] 《 新一代视频压缩编码标准 -- H264/AVC（第二版）》
