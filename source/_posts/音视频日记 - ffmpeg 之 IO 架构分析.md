---
title: 音视频日记 - ffmpeg 之 IO 架构分析
abbrlink: ce615ee4
date: 2021-01-13 13:11:40
categories:
    - 音视频日记
tags:
    - ffmpeg
---
协议 (文件) 操作的顶层结构是 `AVIOContext`，该对象实现了带缓冲的读写操作

![协议操作对象结构](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20210115000338.png)

<!-- more -->

`ffmpeg` 的输入对象 `AVFormatContext` 的 `pb` 字段指向一个 `AVIOContext`

`AVIOContext` 的 `opaque` 实际指向一个 `URLContext` 对象，这个对象封装了协议对象及协议操作对象
    - `prot` 指向具体的协议操作对象 `URLProtocol`
    - `priv_data` 指向具体的协议对象

`URLProtocol` 为协议操作对象，针对每种协议，会有一个这样的对象，每个协议操作对象和一个协议对象关联
    - 比如，文件操作对象为 `ff_file_protocol` 关联的结构体是 `FileContext`

## 函数调用关系

``` c++
+ avformat_open_input()
+---+ init_input()                              /* 打开输入的视频数据并且探测视频的格式 */
    +---- av_probe_input_buffer2()
        +---- avio_read()
        +---- av_probe_input_format2()
            +---- av_probe_input_format3()
    +---+ avio_open2()
        +---+ ffurl_open_whitelist()
            +---+ ffurl_alloc()
                +---- url_find_protocol()
                +---- url_alloc_for_protocol()
            +---+ ffurl_connect()
                +---- prot->url_open2()
                +---- prot->url_open()
        +---+ ffio_fdopen()
            +---- avio_alloc_context()
```

## init_input()

``` C++

/* Open input file and probe the format if necessary. */
static int init_input(AVFormatContext *s, const char *filename,
                      AVDictionary **options)
{
    int ret;
    AVProbeData pd = {filename, NULL, 0};
    int score = AVPROBE_SCORE_RETRY;

    if (s->pb) {
        s->flags |= AVFMT_FLAG_CUSTOM_IO;
        if (!s->iformat)
            return av_probe_input_buffer2(s->pb, &s->iformat, filename,
                                         s, 0, s->format_probesize);
        else if (s->iformat->flags & AVFMT_NOFILE)
            av_log(s, AV_LOG_WARNING, "Custom AVIOContext makes no sense and"
                                      "will be ignored with AVFMT_NOFILE format.\n");
        return 0;
    }

    if ((s->iformat && s->iformat->flags & AVFMT_NOFILE) ||
        (!s->iformat && (s->iformat = av_probe_input_format2(&pd, 0, &score))))
        return score;

    if ((ret = s->io_open(s, &s->pb, filename, AVIO_FLAG_READ | s->avio_flags, options)) < 0)
        return ret;

    if (s->iformat) return 0;
    return av_probe_input_buffer2(s->pb, &s->iformat, filename,
                                 s, 0, s->format_probesize);
}
```

这个函数在短短的几行代码中包含了好几个 `return`，因此逻辑还是有点复杂的，我们可以梳理一下：

在函数的开头的 `score` 变量是一个判决 `AVInputFormat` 的分数的门限值，如果最后得到的 `AVInputFormat` 的分数低于该门限值，就认为没有找到合适的 `AVInputFormat`。`ffmpeg` 内部判断封装格式的原理实际上是对每种 `AVInputFormat` 给出一个分数，满分是 `100` 分，越有可能正确的 `AVInputFormat` 给出的分数就越高。最后选择分数最高的 `AVInputFormat` 作为推测结果。`score` 的值是一个宏定义 `AVPROBE_SCORE_RETRY`，我们可以看一下它的定义：

``` C++
#define AVPROBE_SCORE_MAX       100 ///< maximum score
#define AVPROBE_SCORE_RETRY (AVPROBE_SCORE_MAX/4)
```

由此我们可以得出 `score` 取值是 `25`，即如果推测后得到的最佳 `AVInputFormat` 的分值低于 `25`，就认为没有找到合适的  `AVInputFormat`

整个函数的逻辑大体如下：

1. 当使用了自定义的 `AVIOContext` 的时候（`AVFormatContext` 中的 `AVIOContext` 不为空，即 `s->pb!=NULL`），如果指定了 `AVInputFormat` 就直接返回，如果没有指定就调用 `av_probe_input_buffer2()` 推测 `AVInputFormat`。这一情况出现的不算很多，但是当我们从内存中读取数据的时候（需要初始化自定义的 `AVIOContext`），就会执行这一步骤
2. 在更一般的情况下，如果已经指定了 `AVInputFormat`，就直接返回；如果没有指定 `AVInputFormat`，就调用 `av_probe_input_format(NULL,…)` 根据文件路径判断文件格式。这里特意把 `av_probe_input_format()` 的第 1 个参数写成 `NULL`，是为了强调这个时候实际上并没有给函数提供输入数据，此时仅仅通过文件路径推测 `AVInputFormat`
3. 如果发现通过文件路径判断不出来文件格式，那么就需要打开文件探测文件格式了，这个时候会首先调用 `avio_open2()` 打开文件，然后调用 `av_probe_input_buffer2()` 推测 `AVInputFormat`

## 参考资料

* [1] [2——FFMPEG 之协议 (文件) 操作 ----AVIOContext, URLContext, URLProtocol_finewind 的专栏 - CSDN 博客](https://blog.csdn.net/finewind/article/details/39433055)
