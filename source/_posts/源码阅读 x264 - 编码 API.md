---
title: 源码阅读 x264 - 编码 API
categories:
  - 源码阅读
tags:
  - x264
abbrlink: eb0de708
date: 2020-08-03 13:34:58
---
在 {% post_link "源码阅读 x264 - 命令行工具" %} 中我们分析到 `encode()` 的流程中涉及到 `libx264` 的几个关键的 API，这些函数有：

- `x264_encoder_open()`：打开 H.264 编码器。
- `x264_encoder_headers()`：输出 SPS/PPS/SEI。
- `x264_encoder_encode()`：编码一帧数据。
- `x264_encoder_close()`：关闭 H.264 编码器。

<!-- more -->

## x264_encoder_open

``` cpp
/**
* @brief                            创建一个新的编码 handler
* @param[in]        param           x264_param_t 结构体指针
* @return                           x264_t 结构体指针
                                    # not nullptr   执行成功
*                                   # nullptr       执行失败
* @note:    all parameters from x264_param_t are copied
*/
X264_API x264_t *x264_encoder_open(x264_param_t *param);
```

根据函数调用的顺序，看一下 `x264_encoder_open()` 调用的下面几个函数：

### 参数集初始化

- `x264_sps_init()`：根据输入参数生成 `H.264` 码流的 `SPS` 信息。
- `x264_pps_init()`：根据输入参数生成 `H.264` 码流的 `PPS` 信息。

> 参数集初始化的函数解析见：{% post_link "源码阅读 x264 - SPS & PPS" %}

### 帧内预测初始化

- `x264_predict_16x16_init()` 用于初始化 `Intra16x16` 帧内预测函数
- `x264_predict_4x4_init()` 用于初始化 `Intra4x4` 帧内预测函数

> 帧内预测的相关内容详见：{% post_link "编解码技术：H264 - 帧内预测" %}
> 帧内预测的函数解析详见：{% post_link "源码阅读 x264 - 帧内预测" %}

- `x264_pixel_init()` 用于初始化帧内预测方法选择需要计算的代价函数

> `SAD`、`SATD`、`SSD`、`SSIM` 的介绍详见：{% post_link "编解码技术：H264 - 代价计算" %}
> `SAD`、`SATD`、`SSD`、`SSIM` 的函数解析见：{% post_link "源码阅读 x264 - 代价计算" %}

### 变换参数初始化

- `x264_dct_init()` 初始化了一系列的 `DCT` 变换的函数

> 离散余弦变换的函数解析见：{% post_link "源码阅读 x264 - 离散余弦变换" %}

### 量化参数初始化

- `x264_quant_init()` 函数中初始化了量化有关的函数

> 离散余弦变换的函数解析见：{% post_link "源码阅读 x264 - 离散余弦变换" %}

### 滤波参数初始化

- `x264_deblock_init()`：初始化去块效应滤波器相关的汇编函数

> 离散余弦变换的函数解析见：{% post_link "源码阅读 x264 - 去块效应滤波" %}

## x264_encoder_headers

``` cpp
/* x264_encoder_headers:
 *      return the SPS and PPS that will be used for the whole stream.
 *      *pi_nal is the number of NAL units outputted in pp_nal.
 *      returns the number of bytes in the returned NALs.
 *      returns negative on error.
 *      the payloads of all output NALs are guaranteed to be sequential in memory. */
X264_API int x264_encoder_headers(x264_t *param, x264_nal_t **pp_nal, int *pi_nal);
```

## x264_encoder_encode

``` cpp
/* x264_encoder_encode:
 *      encode one picture.
 *      *pi_nal is the number of NAL units outputted in pp_nal.
 *      returns the number of bytes in the returned NALs.
 *      returns negative on error and zero if no NAL units returned.
 *      the payloads of all output NALs are guaranteed to be sequential in memory. */
X264_API int x264_encoder_encode(x264_t *param, x264_nal_t **pp_nal, int *pi_nal, x264_picture_t *pic_in, x264_picture_t *pic_out);
```

## x264_encoder_close

``` cpp
/* x264_encoder_close:
 *      close an encoder handler */
X264_API void x264_encoder_close(x264_t *param);
```

## 参考资料

* [1] [x264 源代码简单分析：编码器主干部分 - 1 雷霄骅的专栏 - CSDN 博客](https://blog.csdn.net/leixiaohua1020/article/details/45644367)
* [2] [x264 源代码简单分析：编码器主干部分 - 2 雷霄骅的专栏 - CSDN 博客](https://blog.csdn.net/leixiaohua1020/article/details/45719905)
