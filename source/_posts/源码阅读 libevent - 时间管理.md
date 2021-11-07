---
title: 源码阅读 libevent - 时间管理
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 39d8a9e7
date: 2020-12-30 16:24:28
---
为了支持定时器，`libevent` 必须和系统时间打交道，z 在 `event_add` 时，用户制定了一个超时时间，这个超时时间是一个相对时间，但在 `event_add` 函数内部，`libevent` 将其转换为了绝对时间，这就带来了一个问题：在系统时间改变时，`libevent` 需要修改 `event` 的绝对时间，为此 `libevent` 内部有以下几种解决方案：

1. 使用单调时间 `monotonic time`
2. 使用系统时间生成 `fake monotonic time`
3. 使用系统时间（已弃用）

<!--more-->

## 使用 monotonic 时间

### 什么是 monotonic 时间

`monotonic time` 字面意思是单调时间，实际上它指的是系统启动以后流逝的时间，这是由变量 `jiffies` 来记录的。

系统每次启动时 `jiffies` 初始化为 0，每来一个 `timer interrupt`，`jiffies` 加 `1`，也就是说它代表系统启动后流逝的 `tick` 数

### 是否使用 monotonic 时间

决定 `libevent` 是否使用 `monotonic` 时间取决于系统是否支持 `monotonic` 时间，`libevent` 对其有两处判断，一处在编译时判断，一处在运行时判断：

#### 编译时判断

通过 `Cmake` 判断系统环境，来确定获取 `monotonic` 时间的函数。

``` c++
// CMakeLists.txt:
CHECK_FUNCTION_EXISTS_EX(clock_gettime EVENT__HAVE_CLOCK_GETTIME)
CHECK_FUNCTION_EXISTS_EX(mach_absolute_time EVENT__HAVE_MACH_ABSOLUTE_TIME)

// time-internal.h:
#if defined(EVENT__HAVE_CLOCK_GETTIME) && defined(CLOCK_MONOTONIC)
#define HAVE_POSIX_MONOTONIC
#elif defined(EVENT__HAVE_MACH_ABSOLUTE_TIME)
#define HAVE_MACH_MONOTONIC
#elif defined(_WIN32)
#define HAVE_WIN32_MONOTONIC
#else
#define HAVE_FALLBACK_MONOTONIC
#endif
```

根据 `time-internal.h` 中的四个不同的宏定义，`libevent` 定义了四个 `evutil_configure_monotonic_time_` 函数，本文主要分析 `HAVE_POSIX_MONOTONIC` 宏定义下的 `evutil_configure_monotonic_time_` 函数。

#### 运行时判断

`libevent` 会在 `event_base` 创建时对 `monotonic` 时间的支持进行判断，判断方式为：调用不同平台下的 `monotonic` 时间获取函数，如果能正确返回，则代表该平台支持 `monotonic` 时间，`libevent` 后续使用 `gettime` 获取时间均会使用 `monotonic` 时间。

``` c++
/* The POSIX clock_gettime() interface provides a few ways to get at a monotonic clock.
   CLOCK_MONOTONIC is most widely supported.
   Linux also provides a CLOCK_MONOTONIC_COARSE with accuracy of about 1-4 msec.
   On all platforms I'm aware of, CLOCK_MONOTONIC really is monotonic.
   Platforms don't agree about whether it should jump on a sleep/resume. */
int evutil_configure_monotonic_time_(struct evutil_monotonic_timer *base, int flags) {
    /* CLOCK_MONOTONIC exists on FreeBSD, Linux, and Solaris.  You need to check for it at runtime,
     * because some older kernel versions won't have it working. */
    const int fallback = flags & EV_MONOT_FALLBACK;
    struct timespec ts;

    if (!fallback && clock_gettime(CLOCK_MONOTONIC, &ts) == 0) {
        base->monotonic_clock = CLOCK_MONOTONIC;
        return 0;
    }
    if (CLOCK_MONOTONIC < 0) event_errx(1,"I didn't expect CLOCK_MONOTONIC to be < 0");
    base->monotonic_clock = -1;
    return 0;
}
```

如果 `libevent` 所在的系统支持 `monotonic` 时间，那么根本就不用考虑用户手动修改系统时间这坑爹的事情。但如果所在的系统没有支持 `monotonic` 时间，那么 `libevent` 就只能使用 `evutil_gettimeofday` 获取一个用户能修改的时间。

## 使用 fake monotonic 时间

从上述分析中可以看到，在平台不支持 `base->monotonic_clock` 会被 `libevent` 置为 `-1`，此时调用 `gettime` 就会调用 `evutil_gettimeofday` 来获取系统时间，并通过 `adjust_monotonic_time` 将获取到的系统时间调整为一个单调递增的时间。

``` c++
int evutil_gettime_monotonic_(struct evutil_monotonic_timer *base, struct timeval *tp) {
    struct timespec ts;
    if (base->monotonic_clock < 0) {
        if (evutil_gettimeofday(tp, NULL) < 0) return -1; // evutil_gettimeofday 调用 gettimeofday 函数
        adjust_monotonic_time(base, tp);
        return 0;
    }

    if (clock_gettime(base->monotonic_clock, &ts) == -1) return -1;
    tp->tv_sec = ts.tv_sec;
    tp->tv_usec = ts.tv_nsec / 1000;
    return 0;
}

static int gettime(struct event_base *base, struct timeval *tp) {
    if (base->tv_cache.tv_sec) {
        *tp = base->tv_cache;
        return 0;
    }
    if (evutil_gettime_monotonic_(&base->monotonic_timer, tp) == -1) return -1;
    ... ...
    return 0;
}
```

### 系统时间到 fake monotonic 时间的转化

`libevent` 获取到系统时间后，会调用 `adjust_monotonic_time` 生成 `fake monotonic` 时间。

``` c++
/* This function assumes it's called repeatedly with a not-actually-so-monotonic time source whose outputs
   are in 'tv'. It implements a trivial ratcheting mechanism so that the values never go backwards. */
static void adjust_monotonic_time(struct evutil_monotonic_timer *base, struct timeval *tv) {
    /* tv 输入的是 real time，输出为 fake monotonic time */
    evutil_timeradd(tv, &base->adjust_monotonic_clock, tv); // evutil_timeradd(a,b,c) <--> c = a + b
    // 如果 tv < last_time 表明用户向前调整时间了，需要校正时间
    if (evutil_timercmp(tv, &base->last_time, <)) { // evutil_timercmp(a,b,op) <--> a op b
        struct timeval adjust; // 保存差值
        evutil_timersub(&base->last_time, tv, &adjust); // evutil_timersub(a,b,c) <--> c = a - b
        evutil_timeradd(&adjust, &base->adjust_monotonic_clock, &base->adjust_monotonic_clock);
        *tv = base->last_time;
    }
    base->last_time = *tv;
}
```

从上面代码中可以看出，如果用户没有向前调整时间，则 `adjust_monotonic_time` 函数返回的就是真实的时间，如果用户向前调整了时间，`libevent` 会将用户向前调整的时间累计到 `base->adjust_monotonic_clock` 参数中，再之后获取的时间就都会被 `adjust_monotonic_time` 函数加上 `base->adjust_monotonic_clock` 再返回出来，从而实现时间的单调递增，即 `fake monotonic` 时间

## 使用系统时间

该部分内容已被 `libevent` 于 `2012` 年弃用，如读者有兴趣，可以查看 [参考资料[2]](https://blog.csdn.net/luotuo44/article/details/38661787) 中的分析。

``` git
Revision: f5e4eb05e5bdf61b0a1b11f99effaf835a8796ce
Author: Nick Mathewson <nickm@torproject.org>
Date: 2012/4/21 1:14:10
Message:
Refactor monotonic timer handling into a new type and set of functions; add a gettimeofday-based ratcheting implementation
Now, event.c can always assume that we have a monotonic timer; thismakes event.c easier to write.
----
Modified: event-internal.h
Modified: event.c
Modified: evutil_time.c
Modified: time-internal.h
```

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [Libevent 源码分析 - Libevent 时间管理 luotuo44 的专栏 - CSDN 博客](https://blog.csdn.net/luotuo44/article/details/38661787)
