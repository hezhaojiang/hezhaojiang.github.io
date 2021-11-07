---
title: 编解码技术：H264 - Profile
categories:
  - 编解码技术
tags:
  - H.264
abbrlink: 5224fbf9
date: 2020-08-05 13:44:30
---
## H.264 Profiles

2003 年，MPEG 完成了称为 MPEG-4 第 10 部分（H.264）的压缩标准的第一个版本。虽然 H.264 并不是唯一的压缩方法，但它已成为录制、压缩和流式处理高清视频最常用的格式。

H.264 是一个 "系列" 标准，包括许多不同的功能集， 即 profiles。所有的 profile 都严重依赖时态压缩和运动预测来减少帧数。最常用的三个配置文件是 Baseline, Main 和 High。每个 profile 定义用于压缩文件的特定编码技术和算法。

<!-- more -->

### Baseline Profile

#### Baseline Profile 简介

这是最简单的 Profile，主要用于低功耗、低成本设备，包括一些视频会议和移动应用程序。基线配置文件可以达到大约 1000：1 的压缩比，即 1 Gbps 的流可以压缩到大约 1 Mbps。它们使用 4：2：0 色度采样，这意味着颜色信息采样为黑白信息的一半垂直分辨率和水平分辨率的一半。基线配置文件的其他重要功能是使用通用可变长度编码 （UVLC） 和上下文自适应可变长度编码 （CAVLC） 熵编码技术。

#### Baseline Profile 约束

根据 H264 官方文档，Baseline Profile 有如下限制：

- 只有 I 与 P 条带；
- NAL unit 流不应包含取值范围为 2 到 4 的 nal_unit_type，包括 2 与 4；
- 序列参数集中 frame_mbs_only_flag 应为 1；
- 语法元素 chroma_format_idc ， bit_depth_luma_minus8, bit_depth_chroma_minus8, qpprime_y_zero_
transform_bypass_flag，seq_scaling_matrix_present_flag 不应出现在序列参数集中；
- 图像参数集中的参数 weighted_pred_flag 与 weighted_bipred_idc 的取值应该为 0；
- 图像参数集中的参数 entropy_coding_mode_flag 的取值应为 0；
- 图像参数集中 slice_groups_minus1 取值应该在 0 到 7 之间，包括 0 与 7；
- 语法元素 transform_8x8_mode_flag，pic_scaling_matrix_present_flag，与 second_chroma_qp_index_offset
不应出现在图像参数集中；
- 语法元素 level_prefix 的取值不应大于 15；
- 应满足 A.3 节中对于 Baseline Profile 所定义的级别规定。

#### Baseline Profile 配置

Baseline Profile 的 profile_idc 的取值为 66 。

#### Baseline Profile 解码

与 Baseline Profile 某一级别相一致的解码器应可以对所有满足 profile_idc 等于 66 或 constraint_set0 等于 1，且 level_idc 和 constraint_set3_flag 所表征的级别小于或等于该级别的流进行解码。

### Main Profile

#### Main Profile 简介

Main Profile 包括 Baseline Profile 的所有功能，但改进了帧预测算法。它用于使用 MPEG-4 格式的 SD 数字电视广播，但不包括高清广播。

#### Main Profile 约束

根据 H264 官方文档，Main Profile 有如下限制：

- 只有 I、P 和 B 条带；
- NAL unit 流不应包含从 2 到 4 取值的 nal_unit_type，包括 2 与 4；
- 不允许任意的条带顺序；
- 语法元素 hroma_format_idc, bit_depth_luma_minus8, bit_depth_chroma_minus8, qpprime_y_zero_
transform_bypass_flag, 与 seq_scaling_matrix_present_flag 不应在序列参数集中出现。
- 图像参数集中应该具有只等于 0 的 num_slice_groups_minus1；
- 图像参数集中应该具有只等于 0 的 redundant_pic_cnt_present_flag；
- 语法元素 transform_8x8_mode_flag, pic_scaling_matrix_present_flag, 与 second_chroma_qp_index_offset 不应出现在图像参数集中；
- 语法元素 level_prefix 不应大于 15（如果出现时）；
– 语法元素 pcm_sample_luma[i] (i = 0..255)，和 pcm_sample_chroma[i] (i = 0..2 * MbWidthC * MbHeightC − 1) 不应等于 0 (如果出现时)。
- 应满足 A.3 节中对于 Main Profile 所定义的级别的规定。

#### Main Profile 配置

Main Profile 的 profile_idc 的取值为 77 。

#### Main Profile 解码

与 Main Profile 某一级别相一致的解码器应可以对所有满足 profile_idc 等于 77 或 constraint_set1 等于 1，且 level_idc 和 constraint_set3_flag 所表征的级别小于或等于该级别的流进行解码。

### High Profile

#### High Profile 简介

H.264 High Profile 是 H.264 系列中最高效、最强大的配置文件，是广播和光盘存储的主要配置文件，尤其是 HDTV 和 Bluray 光盘存储格式。它可以达到约 2000：1 的压缩比。高轮廓还使用可选择 4x4 或 8x8 像素块的自适应变换。例如，4x4 块用于图片中细节密集的部分，而细节很少的部分使用 8x8 块进行压缩。结果是保持视频图像质量，同时将网络带宽要求降低高达 50%。通过应用 H.264 高配置文件压缩，1 Gbps 流可以压缩到大约 512 Kbps。

#### High Profile 约束

根据 H264 官方文档，High Profile 有如下限制：

- 只有 I，P 与 B 条带；
- NAL unit 流不应包含取值为 2 到 4 的 nal_unit_type 参数，包括 2 和 4；
- 不允许任意的条带顺序；
- 图像参数集中应有只为 0 的 num_slice_groups_minus1 参数；
- 图像参数集中应有只为 0 的 redundant_pic_cnt_present_flag 参数；
- 序列参数集中应有取值为 0 到 1，包括 0 与 1 的 chroma_format_idc 参数；
- 列参数集中应有只为 0 的 bit_depth_luma_minus8 参数；
- 序列参数集中应有只为 0 的 bit_depth_chroma_minus8 参数；
- 序列参数集中应有只为 0 的 qpprime_y_zero_transform_bypass_flag 参数。
- 应满足 A.3 节中规定的 High Profile 的级别限制。

#### High Profile 配置

High Profile 的 profile_idc 的取值为 100 。

#### High Profile 解码

与 High Profile 某一级别相一致的解码器应可以对所有满足下列两个条件之一的流进行解码，：

- Profile_idc 为 77 或 constraint_set1_flag 为 1，并且 level_idc 和 constraint_set3_flag 组合所表征的级别小于或等于该级别的流进行解码。
- profile_idc 为 100 并且 level_idc 所表征的级别小于或等于该级别的流进行解码。

### 其他 Profile

- Extended profile : profile_idc = 88.
- High 10 profile : profile_idc = 110.
- High 4:2:2 profile : profile_idc = 122.
- High 4:4:4 Predictive profile : profile_idc = 244.
- High 10 Intra profile : profile_idc = 100/110.
- High 4:2:2 Intra profile : profile_idc = 100/110/122.
- High 4:4:4 Intra profile : profile_idc = 44/100/110/122/244.
- CAVLC 4:4:4 Intra profile : profile_idc = 44.

## 参考资料

* [1] 《 T-REC-H.264-201906-I.pdf 》
* [2] 《 T-REC-H.264-200503-S.pdf 官方中文版 》
