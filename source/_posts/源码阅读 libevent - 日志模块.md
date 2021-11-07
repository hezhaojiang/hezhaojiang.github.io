---
title: 源码阅读 libevent - 日志模块
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 5f35962e
date: 2020-10-20 14:01:09
---
`libevent` 的日志模块的文件为 `log.c` 和 `log-internal.h`。

从头文件名就可以看出，`libevent` 的日志函数只提供 `libevent` 内部使用，并不提供外部接口。

这些内部日志接口会在运行过程中一些异常情况时被调用，从而将这些异常的情况打印出来，告知用户。而在这些接口不满足用户的需求时，`libevent` 仍提供了回调函数设置接口方便用户自行修改日志模块的处理方式。

<!--more-->

## 日志模块函数

### 函数声明

从 `log-internal.h` 文件中即可看到 `libevnet` 的日志相关函数声明如下：

``` c++
#define EV_CHECK_FMT(a,b) __attribute__((format(printf, a, b)))
#define EV_NORETURN __attribute__((noreturn))
#define EVENT2_EXPORT_SYMBOL __attribute__ ((visibility("default")))

EVENT2_EXPORT_SYMBOL void event_err(int eval, const char *fmt, ...) EV_CHECK_FMT(2,3) EV_NORETURN;
EVENT2_EXPORT_SYMBOL void event_warn(const char *fmt, ...) EV_CHECK_FMT(1,2);
EVENT2_EXPORT_SYMBOL void event_sock_err(int eval, evutil_socket_t sock, const char *fmt, ...) EV_CHECK_FMT(3,4) EV_NORETURN;
EVENT2_EXPORT_SYMBOL void event_sock_warn(evutil_socket_t sock, const char *fmt, ...) EV_CHECK_FMT(2,3);
EVENT2_EXPORT_SYMBOL void event_errx(int eval, const char *fmt, ...) EV_CHECK_FMT(2,3) EV_NORETURN;
EVENT2_EXPORT_SYMBOL void event_warnx(const char *fmt, ...) EV_CHECK_FMT(1,2);
EVENT2_EXPORT_SYMBOL void event_msgx(const char *fmt, ...) EV_CHECK_FMT(1,2);
EVENT2_EXPORT_SYMBOL void event_debugx_(const char *fmt, ...) EV_CHECK_FMT(1,2);

EVENT2_EXPORT_SYMBOL void event_logv_(int severity, const char *errstr, const char *fmt, va_list ap) EV_CHECK_FMT(3,0);
```

### \_\_attribute\_\_

从上述源码中可以发现，所有的日志 API 均被一些宏定义所修饰，这些宏定义有：

``` c++
#define EV_CHECK_FMT(a,b) __attribute__((format(printf, a, b)))
#define EV_NORETURN __attribute__((noreturn))
#define EVENT2_EXPORT_SYMBOL __attribute__ ((visibility("default")))
```

其意义如下：

1. `__attribute__((format(printf, a, b)))`

    `format` 属性可以给被声明的函数加上类似 `printf` 或者 `scanf` 的特征，它可以使编译器检查函数声明和函数实际调用参数之间的格式化字符串是否匹配。`format` 属性告诉编译器，按照 `printf`，`scanf` 等标准 `C` 函数参数格式规则对该函数的参数进行检查。

    如 `EV_CHECK_FMT(2,3);` 就提示编译器按照 `printf` 函数格式化的形式来对函数进行编译，并且从第 `3` 个参数开始按照第 `2` 个参数字符串的格式进行格式化。

2. `__attribute__((noreturn))`

    `noreturn` 属性告诉编译器此函数不会返回（注意：是不会返回，而不是没有返回值）。 `C` 库函数 `abort()` 和 `exit()` 都使用此属性声明。

3. `__attribute__ ((visibility("default")))`

    `visibility` 属性可以控制共享文件导出符号。`"default"`：用它定义的符号将被导出，动态库中的函数默认是可见的。`"hidden"`：用它定义的符号将不被导出，并且不能从其它对象进行使用，动态库中的函数是被隐藏的。`default` 意味着该方法对其它模块是可见的。而 `hidden` 表示该方法符号不会被放到动态符号表里，所以其它模块 (可执行文件或者动态库) 不可以通过符号表访问该方法。

### 函数定义

`libevnet` 的日志 `API` 的函数定义较为简单，下面列出几个示例，其余函数定义可详见源码：

``` c++
void event_err(int eval, const char *fmt, ...) {
    va_list ap;
    va_start(ap, fmt);
    event_logv_(EVENT_LOG_ERR, strerror(errno), fmt, ap);
    va_end(ap);
    event_exit(eval);
}

void event_sock_err(int eval, evutil_socket_t sock, const char *fmt, ...) {
    va_list ap;
    int err = evutil_socket_geterror(sock);
    va_start(ap, fmt);
    event_logv_(EVENT_LOG_ERR, evutil_socket_error_to_string(err), fmt, ap);
    va_end(ap);
    event_exit(eval);
}

void event_debugx_(const char *fmt, ...) {
    va_list ap;
    va_start(ap, fmt);
    event_logv_(EVENT_LOG_DEBUG, NULL, fmt, ap);
    va_end(ap);
}


```

从上述定义中可以发现，这些函数调用的函数为：

1. `event_logv_`
2. `evutil_socket_geterror` 和 `evutil_socket_error_to_string`
3. `event_exit`

#### event_logv_

``` c++
static void event_log(int severity, const char *msg) {
    if (log_fn) log_fn(severity, msg);
    else {
        const char *severity_str;
        switch (severity) {
        case EVENT_LOG_DEBUG:
            severity_str = "debug"; break;
        case EVENT_LOG_MSG:
            severity_str = "msg"; break;
        case EVENT_LOG_WARN:
            severity_str = "warn"; break;
        case EVENT_LOG_ERR:
            severity_str = "err"; break;
        default:
            severity_str = "???"; break;
        }
        (void)fprintf(stderr, "[%s] %s\n", severity_str, msg);
    }
}

void event_logv_(int severity, const char *errstr, const char *fmt, va_list ap) {
    char buf[1024];
    size_t len;

    /* DEBUG 等级日志需要在 DEBUG 打开是才能输出 */
    if (severity == EVENT_LOG_DEBUG && !event_debug_get_logging_mask_()) return;

    if (fmt != NULL) evutil_vsnprintf(buf, sizeof(buf), fmt, ap);
    else buf[0] = '\0';

    if (errstr) {
        len = strlen(buf);
        if (len < sizeof(buf) - 3) evutil_snprintf(buf + len, sizeof(buf) - len, ": %s", errstr);
    }

    event_log(severity, buf);
}
```

从源代码中可以看出 `event_logv_` 的功能较为简单，其根本上是调用 `fprintf` 将错误日志打印到 `stderr`，并且也提供了函数指针 `log_fn` 以供用户自行替换打印日志函数，替换打印日志函数可调用 `event_set_log_callback` 函数实现，该函数的声明位于 `event.h`，实现位于 `log.c`，可以被用户调用：

``` c++
static event_log_cb log_fn = NULL;
typedef void (*event_log_cb)(int severity, const char *msg);
void event_set_log_callback(event_log_cb cb) {
    log_fn = cb;
}：
```

`libevent` 默认的日志处理行为是打印在终端屏幕，这往往不符合我们真正的需求。如果我们想按照自己的方式进行日志处理，那么就可以自定义一个日志处理函数（比如说将错误或警告信息输出到文件中），再将该函数名作为参数调用 `event_set_log_callback` 即可，如果想再恢复默认的日志处理行为，那么再次调用 `event_set_log_callback` 函数传入 `NULL` 即可。

#### evutil_socket_geterror

`evutil_socket_geterror` 函数实现在 `GNU` 编译环境下非常简单，仅仅是对 `errno` 和 `strerror(errno)` 的封装，定义该函数的目的是区分不同平台的不同编译环境，如果各位读者有兴趣可自行学习该函数在 `Window` 平台下的实现。

``` c++
#define evutil_socket_geterror(sock) (errno)
#define evutil_socket_error_to_string(errcode) (strerror(errcode))
```

#### event_exit

``` c++
static void event_exit(int errcode) {
    if (fatal_fn) {
        fatal_fn(errcode);
        exit(errcode);  /* should never be reached */
    } else if (errcode == EVENT_ERR_ABORT_)
        abort();
    else
        exit(errcode);
}
```

`event_exit` 的功能也较为简单，其根本上是调用 `abort()` 或者 `exit()` 退出程序运行。 当然，`event_exit` 也和 `event_logv_` 一样，提供了函数指针 `fatal_fn` 以供用户自行替换退出程序的方式，替换退出程序函数可调用 `event_set_fatal_callback` 函数实现，该函数的声明位于 `event.h`，实现位于 `log.c`，可以被用户调用：

``` c++
typedef void (*event_fatal_cb)(int err);
static event_fatal_cb fatal_fn = NULL;
void event_set_fatal_callback(event_fatal_cb cb) {
    fatal_fn = cb;
}
```

`libevent` 的错误处理仅在发生 `error` 的时候进行，在进行错误处理之前会先进行日志处理，默认的错误处理行为是直接 `abort` 或者 `exit`。如果想在发生错误后，程序退出之前做一些其他处理，那么就可以自定义一个错误处理函数，并将该函数名作为参数调用 `event_set_fatal_callback` 即可，如果想再恢复默认的错误处理行为，那么再次调用 `event_set_fatal_callback` 函数传入 `NULL` 即可。

## 参考资料

* [1] [libevent 源码学习（1）：日志及错误处理_HerofH_的博客 - CSDN 博客](https://blog.csdn.net/qq_28114615/article/details/89194018#%E6%97%A5%E5%BF%97%E5%8F%8A%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B)
* [2] [Libevent 源码分析 - 日志和错误处理_luotuo44 的专栏 - CSDN 博客](https://blog.csdn.net/luotuo44/article/details/38317797)
