---
title: 音视频日记 - ffmpeg 关键结构体
categories:
  - 音视频日记
tags:
  - ffmpeg
abbrlink: e03178b1
date: 2021-01-08 16:53:09
---

- `AVFormatContext` : 封装格式上下文结构体，也是统领全局的结构体，保存了视频文件封装格式相关信息
- `AVInputFormat` : 每种封装格式（例如 FLV, MKV, MP4, AVI）对应一个该结构体
- `AVStream` : 视频文件中每个视频（音频）流对应一个该结构体
- `AVCodecContext` : 编码器上下文结构体，保存了视频（音频）编解码相关信息
- `AVCodec` : 每种视频（音频）编解码器 (例如 H.264 解码器) 对应一个该结构体
- `AVPacket` : 存储一帧压缩编码数据
- `AVFrame` : 存储一帧解码后像素（采样）数据

<!-- more -->

## 协议封装 & 解协议

### AVIOContext

`AVIOContext` 是 `ffmpeg` 管理输入输出数据的结构体

#### AVIOContext 相关函数

``` c++
/**
 * Allocate and initialize an AVIOContext for buffered I/O. It must be later freed with avio_context_free().
 *
 * @param buffer Memory block for input/output operations via AVIOContext.
 *        The buffer must be allocated with av_malloc() and friends.
 *        It may be freed and replaced with a new buffer by libavformat.
 *        AVIOContext.buffer holds the buffer currently in use, which must be later freed with av_free().
 * @param buffer_size The buffer size is very important for performance.
 *        For protocols with fixed blocksize it should be set to this blocksize.
 *        For others a typical size is a cache page, e.g. 4kb.
 * @param write_flag Set to 1 if the buffer should be writable, 0 otherwise.
 * @param opaque An opaque pointer to user-specific data.
 * @param read_packet  A function for refilling the buffer, may be NULL.
 *        For stream protocols, must never return 0 but rather a proper AVERROR code.
 * @param write_packet A function for writing the buffer contents, may be NULL.
 *        The function may not change the input buffers content.
 * @param seek A function for seeking to specified byte position, may be NULL.
 * @return Allocated AVIOContext or NULL on failure.
 */
AVIOContext *avio_alloc_context(
                  unsigned char *buffer,
                  int buffer_size,
                  int write_flag,
                  void *opaque,
                  int (*read_packet)(void *opaque, uint8_t *buf, int buf_size),
                  int (*write_packet)(void *opaque, uint8_t *buf, int buf_size),
                  int64_t (*seek)(void *opaque, int64_t offset, int whence));
```

#### AVIOContext 成员变量

- `unsigned char *buffer;` : 缓存开始位置
- `int buffer_size;` : 缓存大小（默认 `32768`）
- `unsigned char *buf_ptr;` : 当前指针读取到的位置
- `unsigned char *buf_end;` : 缓存结束的位置
- `void *opaque;` : `URLContext` 结构体
- `int64_t pos;` : 文件读取指针位置
- `int eof_reached;` : 在读取失败或读取完成时置位
- `(*read_packet)` : 读取音视频数据的函数
- `(*write_packet)` : 写入音视频数据的函数
- `(*read_pause)` : 暂停或恢复网络流媒体协议的播放

### URLContext & URLProtocol

`URLContext` & `URLProtocol` 并不在 `ffmpeg` 提供的头文件中，而是在源代码中

#### URLContext & URLProtocol 定义

``` c++
typedef struct URLContext {
    const AVClass *av_class;    /**<information for av_log(). Set by url_open(). */
    const struct URLProtocol *prot;
    void *priv_data;
    char *filename;             /**< specified URL */
    int flags;
    int max_packet_size;        /**< if non zero, the stream is packetized with this max packet size */
    int is_streamed;            /**<true if streamed (no seek possible), default = false */
    int is_connected;
    AVIOInterruptCB interrupt_callback;
    int64_t rw_timeout;         /**<maximum time to wait for (network) read/write operation completion, in mcs */
    const char *protocol_whitelist;
    const char *protocol_blacklist;
    int min_packet_size;        /**< if non zero, the stream is packetized with this min packet size */
} URLContext;

typedef struct URLProtocol {
    const char *name;
    int     (*url_open)( URLContext *h, const char *url, int flags);
    /**
     * This callback is to be used by protocols which open further nested
     * protocols. options are then to be passed to ffurl_open()/ffurl_connect()
     * for those nested protocols.
     */
    int     (*url_open2)(URLContext *h, const char *url, int flags, AVDictionary **options);
    int     (*url_accept)(URLContext *s, URLContext **c);
    int     (*url_handshake)(URLContext *c);

    /**
     * Read data from the protocol.
     * If data is immediately available (even less than size), EOF is
     * reached or an error occurs (including EINTR), return immediately.
     * Otherwise:
     * In non-blocking mode, return AVERROR(EAGAIN) immediately.
     * In blocking mode, wait for data/EOF/error with a short timeout (0.1s),
     * and return AVERROR(EAGAIN) on timeout.
     * Checking interrupt_callback, looping on EINTR and EAGAIN and until
     * enough data has been read is left to the calling function; see
     * retry_transfer_wrapper in avio.c.
     */
    int     (*url_read) (URLContext *h, unsigned char *buf, int size);
    int     (*url_write)(URLContext *h, const unsigned char *buf, int size);
    int64_t (*url_seek) (URLContext *h, int64_t pos, int whence);
    int     (*url_close)(URLContext *h);
    int (*url_read_pause)(URLContext *h, int pause);
    int64_t (*url_read_seek)(URLContext *h, int stream_index,
                             int64_t timestamp, int flags);
    int (*url_get_file_handle)(URLContext *h);
    int (*url_get_multi_file_handle)(URLContext *h, int **handles,
                                     int *numhandles);
    int (*url_get_short_seek)(URLContext *h);
    int (*url_shutdown)(URLContext *h, int flags);
    int priv_data_size;
    const AVClass *priv_data_class;
    int flags;
    int (*url_check)(URLContext *h, int mask);
    int (*url_open_dir)(URLContext *h);
    int (*url_read_dir)(URLContext *h, AVIODirEntry **next);
    int (*url_close_dir)(URLContext *h);
    int (*url_delete)(URLContext *h);
    int (*url_move)(URLContext *h_src, URLContext *h_dst);
    const char *default_whitelist;
} URLProtocol;
```

在 `URLProtocol` 结构体中，除了一些回调函数接口之外，有一个变量 `const char *name`，该变量存储了协议的名称。每一种输入协议都对应这样一个结构体对象。比如说，文件协议中代码如下：

``` c++
// file.c
const URLProtocol ff_file_protocol = {
    .name                = "file",
    .url_open            = file_open,
    .url_read            = file_read,
    .url_write           = file_write,
    .url_seek            = file_seek,
    .url_close           = file_close,
    .url_get_file_handle = file_get_handle,
    .url_check           = file_check,
    .url_delete          = file_delete,
    .url_move            = file_move,
    .priv_data_size      = sizeof(FileContext),
    .priv_data_class     = &file_class,
    .url_open_dir        = file_open_dir,
    .url_read_dir        = file_read_dir,
    .url_close_dir       = file_close_dir,
    .default_whitelist   = "file,crypto,data"
};
```

## 封装 & 解封装

### AVFormatContext

#### AVFormatContext 相关函数

``` c++
// 分配一个 AVFormatContext
AVFormatContext *avformat_alloc_context(void);
// 释放 AVFormatContext 及其所有 AVStream
void avformat_free_context(AVFormatContext *s);
// 打开一个输入流并读取报头，该函数不会打开编解码器
int avformat_open_input(AVFormatContext **ps, const char *url, ff_const59 AVInputFormat *fmt, AVDictionary **options);
// 读取媒体文件的数据包以获取流信息
int avformat_find_stream_info(AVFormatContext *ic, AVDictionary **options);
```

#### AVFormatContext 成员变量

- `struct AVInputFormat *iformat;` : 输入音视频的 `AVInputFormat`
- `struct AVOutputFormat *oformat;` : 输出音视频的 `AVInputFormat`
- `AVIOContext *pb;` : 输入 / 输出音视频数据的缓存
- `int64_t duration;` : 输入 / 输出音视频的时长（以微秒为单位）
- `int64_t bit_rate;` : 输入 / 输出音视频的码率
- `unsigned int nb_streams;` ： 输入 / 输出音视频的 `AVStream` 个数
- `AVStream **streams;` ： 输入 / 输出音视频的每个视频（音频流）
- `AVDictionary *metadata;` ： 元数据

### AVStream

视频文件中每个视频（音频）流对应一个 `AVStream` 对象

- 在解封装时，`AVStream` 对象由 `libavformat` 中的函数 `xxxx_read_header()` 调用 `avformat_open_input()` 创建
- 如果解封装时，未在文件中读到文件头，即标志位 `AVFMTCTX_NOHEADER` 被置位，由函数 `xxxx_read_packet()` 调用 `avformat_open_input()` 创建
- 在封装时，`AVStream` 对象由用户调用函数 `avformat_new_stream()` 创建
- 调用 `libavformat` 中的函数 `avformat_free_context()` 销毁对象

#### AVStream 相关函数

``` c++
/**
 * Add a new stream to a media file.
 * @param s media file handle
 * @param c If non-NULL, the AVCodecContext corresponding to the new stream will be initialized to use this codec.
 * This is needed for e.g. codec-specific defaults to be set, so codec should be provided if it is known.
 * @return newly created stream or NULL on error.
 */
AVStream *avformat_new_stream(AVFormatContext *s, const AVCodec *c)
```

#### AVStream 成员变量

- `int index` : stream index in AVFormatContext
- `AVCodecContext *codec;` : 指向该视频 / 音频流的 AVCodecContext 它们是一一对应的关系
- `AVRational time_base;` : 该流的时基 通过该值可以把 PTS/DTS 转化为真正的时间
- `int64_t duration;` : 该视频 / 音频流长度
- `AVRational r_frame_rate;` : 该流的帧率
- `AVDictionary *metadata;` : 元数据信息
- `AVRational avg_frame_rate;` : 平均帧率（注 : 对视频来说，这个挺重要的）
- `AVPacket attached_pic;` : 附带的图片。比如说一些 MP3，AAC 音频文件附带的专辑封面
- `AVCodecParameters *codecpar;` : 与此流相关的编解码器参数

``` c++
// AVRational 可以用来表示无理数：
    typedef struct AVRational {
    int num; ///< Numerator 分子
    int den; ///< Denominator 分母
} AVRational;
```

### AVInputFormat

该结构被称为 `demuxer`，是音视频文件的一个解封装器

> `AVOutputFormat` 和 `AVInputFormat` 结构类似，是音视频文件的封装器

#### AVInputFormat 相关函数

``` c++
/**
 * If f is NULL, returns the first registered input format,
 * if f is non-NULL, returns the next registered input format after f or NULL if f is the last one.
 */
attribute_deprecated AVInputFormat  *av_iformat_next(const AVInputFormat  *f);
```

遍历 `ffmpeg` 中的解封装器的方法：

1. 注册所有解封装器：`av_register_all();`。注意，需要调用该函数才可以将所有注册的解封装器连接为一个链表
2. 声明一个 `AVInputFormat` 类型的指针，比如说 `AVInputFormat* first_f = nullptr;`
3. 调用 `av_iformat_next()` 函数，即可获得指向链表下一个解封装器的指针，循环往复可以获得所有解码器的信息

#### AVInputFormat 成员变量

每种文件格式对应了一个 `AVInputFormat` 对象，这些对象中的成员变量基本为 `const` 类型

- `const char *name;` : 封装格式名称
- `const char *long_name;` : 封装格式的长名称
- `const char *extensions;` : 封装格式的扩展名
- `int raw_codec_id;` : 封装格式 `ID`
- 一些封装格式处理的接口回调函数，如 `read_header`, `read_packet`, `read_close`, `read_seek` 等

### 编码 & 解码

### AVCodecContext

### AVCodecContext 相关函数

``` c++
// 分配一个 AVCodecContext，并将其字段设置为默认值
AVCodecContext *avcodec_alloc_context3(const AVCodec *codec);
// 释放 AVCodecContext 及其相关内容，并向提供的指针写入 nullptr
void avcodec_free_context(AVCodecContext **avctx);
// 关闭一个给定的 AVCodecContext 并释放所有与它相关的数据
int avcodec_close(AVCodecContext *avctx);
/**
 * Fill the codec context based on the values from the supplied codec parameters.
 * Any allocated fields in codec that have a corresponding field in par are
 * freed and replaced with duplicates of the corresponding field in par.
 * Fields in codec that do not have a counterpart in par are not touched.
 * @return >= 0 on success, a negative AVERROR code on failure.
 */
int avcodec_parameters_to_context(AVCodecContext *codec, const AVCodecParameters *par);
```

#### AVCodecContext 成员变量

- `const struct AVCodec  *codec;` : 编解码器的 `AVCodec`
- `int width, height;` : 图像的宽高（只针对视频）
- `int coded_width, coded_height;` 未裁剪的图像宽高
- `enum AVPixelFormat pix_fmt;` : 像素格式（只针对视频）
- `AVRational time_base;` : 帧率的信息
- `int sample_rate;` : 采样率（只针对音频）
- `int channels;` : 采样率（只针对音频）
- `enum AVSampleFormat sample_fmt;` : 采样格式（只针对音频）

``` linux
    **** width/coded_width & height/coded_height ****
一些编码器要求帧尺寸是特定数字的倍数，例如 x264 的帧尺寸是 16 的倍数
因此，如果需要，编码器将把帧填充到合适的数字，并为解码器存储裁剪值
coded_width, coded_height 是裁剪前的大小
width, height 是裁剪后的大小
```

### AVCodec

AVCodec 是存储编解码器信息的结构体

#### AVCodec 相关函数

``` c++
// 如果 c 为空，返回第一个注册的编解码器
// 如果 c 非空，返回 c 之后的下一个注册编解码器，如果 c 是最后一个编解码器，返回 nullptr
attribute_deprecated AVCodec *av_codec_next(const AVCodec *c);
// 查找具有匹配编解码器 ID 的已注册解码器
AVCodec *avcodec_find_decoder(enum AVCodecID id);
// 初始化 AVCodecContext 使用给定的 AVCodec。在使用这个函数之前，必须使用 avcodec_alloc_context3() 分配上下文。
int avcodec_open2(AVCodecContext *avctx, const AVCodec *codec, AVDictionary **options);
```

遍历 `ffmpeg` 中的解码器信息的方法：

1. 注册所有编解码器：`av_register_all();`。注意，需要调用该函数才可以将所有注册的解码器连接为一个链表
2. 声明一个 `AVCodec` 类型的指针，比如说 `AVCodec* first_c = nullptr;`
3. 调用 `av_codec_next()` 函数，即可获得指向链表下一个解码器的指针，循环往复可以获得所有解码器的信息。注意，如果想要获得指向第一个解码器的指针，则需要将该函数的参数设置为 `nullptr`

#### AVCodec 成员变量

- `const char *name;` : 编解码器的名字，比较短
- `const char *long_name;` : 编解码器的名字，全称，比较长
- `enum AVMediaType type;` : 指明了类型，是视频，音频，还是字幕
- `enum AVCodecID id;` : ID，不重复
- `const AVRational *supported_framerates;` : 支持的帧率（仅视频）
- `const enum AVPixelFormat *pix_fmts;` : 支持的像素格式（仅视频）
- `const int *supported_samplerates;` : 支持的采样率（仅音频）
- `const enum AVSampleFormat *sample_fmts;` : 支持的采样格式（仅音频）
- `const uint64_t *channel_layouts;` : 支持的声道数（仅音频）
- `int priv_data_size;` : 私有数据的大小

每一个编解码器对应一个该结构体，查看一下 `ffmpeg` 的源代码，我们可以看一下 `H.264` 解码器的结构体如下所示：

``` c++
// h264.c
AVCodec ff_h264_decoder = {
    .name                  = "h264",
    .long_name             = NULL_IF_CONFIG_SMALL("H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10"),
    .type                  = AVMEDIA_TYPE_VIDEO,
    .id                    = AV_CODEC_ID_H264,
    .priv_data_size        = sizeof(H264Context),
    .init                  = h264_decode_init,
    .close                 = h264_decode_end,
    .decode                = h264_decode_frame,
    .capabilities          = /*AV_CODEC_CAP_DRAW_HORIZ_BAND |*/ AV_CODEC_CAP_DR1 |
                             AV_CODEC_CAP_DELAY | AV_CODEC_CAP_SLICE_THREADS |
                             AV_CODEC_CAP_FRAME_THREADS,
    .hw_configs            = (const AVCodecHWConfigInternal *const []) { NULL },
    .caps_internal         = FF_CODEC_CAP_INIT_THREADSAFE | FF_CODEC_CAP_EXPORTS_CROPPING |
                             FF_CODEC_CAP_ALLOCATE_PROGRESS | FF_CODEC_CAP_INIT_CLEANUP,
    .flush                 = h264_decode_flush,
    .update_thread_context = ONLY_IF_THREADS_ENABLED(ff_h264_update_thread_context),
    .profiles              = NULL_IF_CONFIG_SMALL(ff_h264_profiles),
    .priv_class            = &h264_class,
};
```

## 音视频数据

### AVPacket

`AVPacket` 是存储压缩编码数据相关信息的结构体

#### AVPacket 相关函数

``` c++
// 返回流的下一帧
int av_read_frame(AVFormatContext *s, AVPacket *pkt);
// 向解码器提供原始数据包数据作为输入
int avcodec_send_packet(AVCodecContext *avctx, const AVPacket *avpkt);
```

#### AVPacket 结构体定义

``` c++
typedef struct AVPacket {
    AVBufferRef *buf;
    int64_t pts;    ///< 显示时间戳
    int64_t dts;    ///< 解码时间戳
    uint8_t *data;  ///< 压缩编码数据
    int   size;     ///< 压缩编码数据大小
    int   stream_index; ///< 所属的 AVStream
    int   flags;    ///< A combination of AV_PKT_FLAG values
    AVPacketSideData *side_data;
    int side_data_elems;
    int64_t duration;
    int64_t pos;    ///< byte position in stream, -1 if unknown
} AVPacket;
```

``` linux
   *** 对于 H.264 来说。1 个 AVPacket 的 data 通常对应一个 NALU ***
通过查看 ffmpeg 源代码我们发现，AVPacket 中的数据起始处没有分隔符 0x00000001
也不是 0x65、0x67、0x68、0x41 等字节
所以 AVPacket 肯定这不是标准的 NALU
其实，AVPacket 前 4 个字表示的是 NALU 的长度，从第 5 个字节开始才是 NALU 的数据
所以直接将 AVPacket 前 4 个字节替换为 0x00000001 即可得到标准的 NALU 数据
```

### AVFrame

#### AVFrame 相关函数

``` c++
// 分配 AVFrame 并将其字段设置为默认值
AVFrame *av_frame_alloc(void);
// 释放 AVFrame 和其中所有动态分配的成员
void av_frame_free(AVFrame **frame);
// 从解码器返回已解码的输出数据
int avcodec_receive_frame(AVCodecContext *avctx, AVFrame *frame);
```

#### AVPacket 成员变量

``` c++
typedef struct AVFrame {
#define AV_NUM_DATA_POINTERS 8
    uint8_t *data[AV_NUM_DATA_POINTERS];    // 解码后原始数据 对视频来说是 YUV，RGB，对音频来说是 PCM
    int linesize[AV_NUM_DATA_POINTERS];     // data 中一行数据的大小。注意：未必等于图像的宽，一般大于图像的宽
    uint8_t **extended_data;
    int width, height;  // 视频帧宽和高（1920x1080, 1280x720 ...）
    int nb_samples;     // 音频的一个 AVFrame 中可能包含多个音频帧，在此标记包含了几个
    int format;         // 解码后原始数据类型（YUV420, YUV422, RGB24 ...）
    int key_frame;      // 1 -> keyframe, 0-> not
    enum AVPictureType pict_type;   // 帧类型（I, B, P ...）
    AVRational sample_aspect_ratio; // 宽高比（16:9, 4:3 ...）
    int64_t pts;        // 显示时间戳
    int64_t pkt_dts;
    int coded_picture_number;   // 编码帧序号
    int display_picture_number; // 显示帧序号
    ... ...
} AVFrame;
```

## 参考资料

* [1] 基于 FFmpeg + SDL 的视频播放器的制作 -- 雷霄骅
* [2] [FFMPEG 中最关键的结构体之间的关系_雷霄骅 (leixiaohua1020) 的专栏 - CSDN 博客_ffmpeg 结构体](https://blog.csdn.net/leixiaohua1020/article/details/11693997)
