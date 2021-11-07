---
title: 编解码技术：H264 - Level
categories:
  - 编解码技术
tags:
  - H.264
abbrlink: 3552c034
date: 2020-08-05 14:46:33
---
## H.264 Level

在 H.264 标准中，有许多不同的 "级别" 指定约束，指示配置文件所需的解码器性能程度，这些 "级别" 即为 Level。Level 指定设备可以播放的最大数据速率和视频分辨率。

例如，配置文件中的支持级别将指定解码器可能能够使用的最大图像分辨率、帧速率和比特率。较低的级别意味着较低的分辨率、允许的最大比特率较低以及存储参考帧的内存要求更低。需要符合给定级别的解码器才能解码为该级别和所有较低级别编码的所有位流。

<!-- more -->

### 基础通用 Level 限制

基础通用 Level 限制的适用于 Baseline, Main and Extended profiles，下表该说明了通用 Level 限制：

- 如果表条目标记为 "-"，则变量的值不受限制，这是在指定 Level 对 Profile 进行位流一致性的要求。
- 否则，表条目将为相关限制指定变量的值，该限制是对 Level 符合 Profile 的级别的要求。

为 Level 能力比较的目的，如果一个 Level 比另一 Level 更接近于下表的表头（表底），该 Level 应认为比另一 Level 更低（高）。

与比特流相一致的 Level 应在语法元素 level_idc 与 constraint_set3_flag 里指明，如下所示：

- 如果 level_idc 等于 11 且 constraint_set3_flag 也等于 1，表明 Level 为 1b 级
- 否则（level_idc 不等于 11，或 constraint_set3_flag 不等于 1），level_idc 应设为等于下表中 Level 号的 10 倍的值，且 constraint_set3_flag 应设为 0。

![Level limits](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200809221330.jpg)

### 其他通用 Level 限制

其他通用 Level 限制的适用于 High, Progressive High, Constrained High, High 10, Progressive High 10,High 4:2:2, High 4:4:4 Predictive, High 10 Intra, High 4:2:2 Intra, High 4:4:4 Intra, and CAVLC 4:4:4 Intra profiles，变量 level_idc 指示比特流符合指定级别：

– 如果 level_idc 等于 9, 表明 Level 为 1b 级
– 否则（level_idc 不等于 9）, level_idc 应设为等于上表中 Level 号的 10 倍的值。

## 不同 Level 最大帧率

不同 Level 最大帧率举例：

![不同 Level 最大帧率举例](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20200805232842.jpg)
