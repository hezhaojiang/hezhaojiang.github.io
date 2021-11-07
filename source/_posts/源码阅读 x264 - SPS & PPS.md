---
title: 源码阅读 x264 - SPS & PPS
categories:
  - 源码阅读
tags:
  - x264
abbrlink: 2633a00a
date: 2021-01-28 15:34:00
---
`x264` 中参数集初始化主要包括如下两个函数：

- `x264_sps_init()`：根据输入参数生成 `H.264` 码流的 `SPS` 信息。
- `x264_pps_init()`：根据输入参数生成 `H.264` 码流的 `PPS` 信息。

<!-- more -->

### x264_sps_init

`x264_sps_init()` 会根据 `x264_param_t` 结构体中的参数来设置 `x264_sps_t` 结构体中的参数。

具体代码及注释如下：

``` c++
/**
* @brief                            初始化 SPS
* @param[out]       sps             x264_sps_t 结构体指针
* @param[in]        i_id            i_id
* @param[in]        param           x264_param_t 结构体指针
* @return                           x264_t 结构体指针
                                    # not nullptr   执行成功
*                                   # nullptr       执行失败
*/
void x264_sps_init(x264_sps_t *sps, int i_id, x264_param_t *param)
{
    int csp = param->i_csp & X264_CSP_MASK;

    sps->i_id = i_id;
    /* 以宏块为单位的图像宽度 */
    sps->i_mb_width = (param->i_width + 15 ) / 16;
    /* 以宏块为单位的图像高度 */
    sps->i_mb_height= (param->i_height + 15 ) / 16;
    /* 所有图像均使用帧编码 */
    sps->b_frame_mbs_only = !(param->b_interlaced || param->b_fake_interlaced);
    if(!sps->b_frame_mbs_only )
        /* 场编码时 以宏块为单位的图像高度为偶数 */
        sps->i_mb_height = (sps->i_mb_height + 1 ) & ~1;
    /* 色度采样结构 yuv444 yuv422 yuv420 yuv400 */
    sps->i_chroma_format_idc = csp >= X264_CSP_I444 ? CHROMA_444 :
                               csp >= X264_CSP_I422 ? CHROMA_422 :
                               csp >= X264_CSP_I420 ? CHROMA_420 : CHROMA_400;

    sps->b_qpprime_y_zero_transform_bypass = param->rc.i_rc_method == X264_RC_CQP && param->rc.i_qp_constant == 0;

    /* 根据其他质量参数 设置 profile */
    if(sps->b_qpprime_y_zero_transform_bypass || sps->i_chroma_format_idc == CHROMA_444 )
        sps->i_profile_idc  = PROFILE_HIGH444_PREDICTIVE;
    else if(sps->i_chroma_format_idc == CHROMA_422 )
        sps->i_profile_idc  = PROFILE_HIGH422;
    else if(BIT_DEPTH> 8 )
        sps->i_profile_idc  = PROFILE_HIGH10;
    else if(param->analyse.b_transform_8x8 || param->i_cqm_preset != X264_CQM_FLAT || sps->i_chroma_format_idc == CHROMA_400 )
        sps->i_profile_idc  = PROFILE_HIGH;
    else if(param->b_cabac || param->i_bframe > 0 || param->b_interlaced || param->b_fake_interlaced || param->analyse.i_weighted_pred > 0 )
        sps->i_profile_idc  = PROFILE_MAIN;
    else
        sps->i_profile_idc  = PROFILE_BASELINE;

    /* constraint_set0 表示编码视频序列遵守 Baseline profile 的所有约束 */
    sps->b_constraint_set0  = sps->i_profile_idc == PROFILE_BASELINE;
    /* constraint_set1 表示编码视频序列遵守 Main profile 的所有约束 */
    /* x264 不支持 baseline 中存在，却不存在 main 中的约束，包括 arbitrary_slice_order 和 slice_groups */
    sps->b_constraint_set1  = sps->i_profile_idc <= PROFILE_MAIN;
    /* Never set constraint_set2, it is not necessary and not used in real world. */
    sps->b_constraint_set2  = 0;
    sps->b_constraint_set3  = 0;

    sps->i_level_idc = param->i_level_idc;
    if(param->i_level_idc == 9 && ( sps->i_profile_idc == PROFILE_BASELINE || sps->i_profile_idc == PROFILE_MAIN ) )
    {
        /* 在一些高级 profile 情况下，level_idc 值为 9 代表等级为 1b */
        /* 在 Baseline or Main profile，level_idc 值为 11 且 constraint_set3 值为 1 代表等级为 1b，（level_idc 值为 11 且 constraint_set3 值为 0 代表等级为 1.1）*/
        sps->b_constraint_set3 = 1;
        sps->i_level_idc      = 11;
    }
    /* Intra profiles */
    if(param->i_keyint_max == 1 && sps->i_profile_idc >= PROFILE_HIGH )
        sps->b_constraint_set3 = 1;

    sps->vui.i_num_reorder_frames = param->i_bframe_pyramid ? 2 : param->i_bframe ? 1 : 0;
    /* extra slot with pyramid so that we don't have to override the
     * order of forgetting old pictures */
    sps->vui.i_max_dec_frame_buffering =
    sps->i_num_ref_frames = X264_MIN(X264_REF_MAX, X264_MAX4(param->i_frame_reference, 1 + sps->vui.i_num_reorder_frames,
                            param->i_bframe_pyramid ? 4 : 1, param->i_dpb_size));
    sps->i_num_ref_frames -= param->i_bframe_pyramid == X264_B_PYRAMID_STRICT;
    if(param->i_keyint_max == 1 )
    {
        sps->i_num_ref_frames = 0;
        sps->vui.i_max_dec_frame_buffering = 0;
    }

    /* number of refs + current frame */
    int max_frame_num = sps->vui.i_max_dec_frame_buffering * (!!param->i_bframe_pyramid+1) + 1;
    /* Intra refresh cannot write a recovery time greater than max frame num-1 */
    if(param->b_intra_refresh )
    {
        int time_to_recovery = X264_MIN(sps->i_mb_width - 1, param->i_keyint_max ) + param->i_bframe - 1;
        max_frame_num = X264_MAX(max_frame_num, time_to_recovery+1);
    }

    sps->i_log2_max_frame_num = 4;
    while((1 << sps->i_log2_max_frame_num) <= max_frame_num )
        sps->i_log2_max_frame_num++;

    sps->i_poc_type = param->i_bframe || param->b_interlaced || param->i_avcintra_class ? 0 : 2;
    if(sps->i_poc_type == 0 )
    {
        int max_delta_poc = (param->i_bframe + 2) * (!!param->i_bframe_pyramid + 1) * 2;
        sps->i_log2_max_poc_lsb = 4;
        while((1 << sps->i_log2_max_poc_lsb) <= max_delta_poc * 2 )
            sps->i_log2_max_poc_lsb++;
    }

    /* 默认带有 VUI 信息 */
    sps->b_vui = 1;

    sps->b_gaps_in_frame_num_value_allowed = 0;
    sps->b_mb_adaptive_frame_field = param->b_interlaced;
    sps->b_direct8x8_inference = 1;

    x264_sps_init_reconfigurable(sps, param);

    sps->vui.b_overscan_info_present = param->vui.i_overscan > 0 && param->vui.i_overscan <= 2;
    if(sps->vui.b_overscan_info_present )
        sps->vui.b_overscan_info = (param->vui.i_overscan == 2 ? 1 : 0 );

    sps->vui.b_signal_type_present = 0;
    sps->vui.i_vidformat = (param->vui.i_vidformat >= 0 && param->vui.i_vidformat <= 5 ? param->vui.i_vidformat : 5 );
    sps->vui.b_fullrange = (param->vui.b_fullrange >= 0 && param->vui.b_fullrange <= 1 ? param->vui.b_fullrange :
                           (csp>= X264_CSP_BGR ? 1 : 0 ) );
    sps->vui.b_color_description_present = 0;

    sps->vui.i_colorprim = (param->vui.i_colorprim >= 0 && param->vui.i_colorprim <= 12 ? param->vui.i_colorprim : 2 );
    sps->vui.i_transfer  = (param->vui.i_transfer  >= 0 && param->vui.i_transfer  <= 18 ? param->vui.i_transfer  : 2 );
    sps->vui.i_colmatrix = (param->vui.i_colmatrix >= 0 && param->vui.i_colmatrix <= 14 ? param->vui.i_colmatrix :
                           (csp>= X264_CSP_BGR ? 0 : 2 ) );
    if(sps->vui.i_colorprim != 2 || sps->vui.i_transfer != 2 || sps->vui.i_colmatrix != 2 )
        sps->vui.b_color_description_present = 1;

    if(sps->vui.i_vidformat != 5 || sps->vui.b_fullrange || sps->vui.b_color_description_present )
        sps->vui.b_signal_type_present = 1;

    /* FIXME: not sufficient for interlaced video */
    sps->vui.b_chroma_loc_info_present = param->vui.i_chroma_loc > 0 && param->vui.i_chroma_loc <= 5 &&
                                         sps->i_chroma_format_idc == CHROMA_420;
    if(sps->vui.b_chroma_loc_info_present )
    {
        sps->vui.i_chroma_loc_top = param->vui.i_chroma_loc;
        sps->vui.i_chroma_loc_bottom = param->vui.i_chroma_loc;
    }

    sps->vui.b_timing_info_present = param->i_timebase_num > 0 && param->i_timebase_den > 0;

    if(sps->vui.b_timing_info_present )
    {
        sps->vui.i_num_units_in_tick = param->i_timebase_num;
        sps->vui.i_time_scale = param->i_timebase_den * 2;
        sps->vui.b_fixed_frame_rate = !param->b_vfr_input;
    }

    sps->vui.b_vcl_hrd_parameters_present = 0; // we don't support VCL HRD
    sps->vui.b_nal_hrd_parameters_present = !!param->i_nal_hrd;
    sps->vui.b_pic_struct_present = param->b_pic_struct;

    // NOTE: HRD related parts of the SPS are initialised in x264_ratecontrol_init_reconfigurable

    sps->vui.b_bitstream_restriction = !(sps->b_constraint_set3 && sps->i_profile_idc >= PROFILE_HIGH);
    if(sps->vui.b_bitstream_restriction )
    {
        sps->vui.b_motion_vectors_over_pic_boundaries = 1;
        sps->vui.i_max_bytes_per_pic_denom = 0;
        sps->vui.i_max_bits_per_mb_denom = 0;
        sps->vui.i_log2_max_mv_length_horizontal =
        sps->vui.i_log2_max_mv_length_vertical = (int)log2f( X264_MAX( 1, param->analyse.i_mv_range*4-1 ) ) + 1;
    }

    sps->b_avcintra = !!param->i_avcintra_class;
    sps->i_cqm_preset = param->i_cqm_preset;
}
```

### x264_pps_init

`x264_pps_init()` 会根据 `x264_param_t` 结构体和 `x264_sps_t` 结构体对 `x264_pps_t` 进行设置。

具体代码及注释如下：

``` c++
/**
* @brief                            初始化 SPS
* @param[out]       sps             x264_sps_t 结构体指针
* @param[in]        i_id            i_id
* @param[in]        param           x264_param_t 结构体指针
* @param[in]        param           x264_sps_t 结构体指针
* @return                           x264_t 结构体指针
                                    # not nullptr   执行成功
*                                   # nullptr       执行失败
*/
void x264_pps_init(x264_pps_t *pps, int i_id, x264_param_t *param, x264_sps_t *sps)
{
    pps->i_id = i_id;
    pps->i_sps_id = sps->i_id;
    pps->b_cabac = param->b_cabac;

    pps->b_pic_order = !param->i_avcintra_class && param->b_interlaced;
    pps->i_num_slice_groups = 1;

    pps->i_num_ref_idx_l0_default_active = param->i_frame_reference;
    pps->i_num_ref_idx_l1_default_active = 1;

    pps->b_weighted_pred = param->analyse.i_weighted_pred > 0;
    pps->b_weighted_bipred = param->analyse.b_weighted_bipred ? 2 : 0;

    pps->i_pic_init_qp = param->rc.i_rc_method == X264_RC_ABR || param->b_stitchable ? 26 + QP_BD_OFFSET : SPEC_QP(param->rc.i_qp_constant );
    pps->i_pic_init_qs = 26 + QP_BD_OFFSET;

    pps->i_chroma_qp_index_offset = param->analyse.i_chroma_qp_offset;
    pps->b_deblocking_filter_control = 1;
    pps->b_constrained_intra_pred = param->b_constrained_intra;
    pps->b_redundant_pic_cnt = 0;

    pps->b_transform_8x8_mode = param->analyse.b_transform_8x8 ? 1 : 0;
}
```
