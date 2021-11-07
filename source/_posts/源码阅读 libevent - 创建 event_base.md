---
title: 源码阅读 libevent - 创建 event_base
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 1a2a4fe3
date: 2020-10-26 15:33:30
---
要实现 `libevent` 的事件处理，最关键的就是 `event_base`，`event_base` 就像是一棵树，而需要进行处理的事件 `event` 就像是树上的果子，因此，在分析 `libevent` 的事件处理之前，先来分析一下 `event_base`。

由于 `event_base` 结构体中包含了三十多个成员，直接去分析每个成员是在当前没有对了 libevent 代码做足够深入理解的话是很难分析每个成员的作用，因此我们先从 `event_base` 的使用中去了解 `event_base` 各个结构体成员的作用会比较好，本文先来看看 `event_base` 的创建。

> {% post_link "源码阅读 libevent - 结构体：event_base" %}

<!--more-->

## 默认方式创建 event_base

通常我们获取 `event_base` 都是通过 `event_base_new` 这个无参函数。使用这个无参函数，只能得到一个默认配置的 `event_base` 结构体。

`event_base_new` 函数的定义如下：

``` cpp
struct event_base * event_base_new(void) {
    struct event_base *base = NULL;
    struct event_config *cfg = event_config_new();
    if (cfg) {
        base = event_base_new_with_config(cfg);
        event_config_free(cfg);
    }
    return base;
}
```

可以看到，其先创建了一个 `event_config` 结构体，并用 `cfg` 指针指向之，然后再用这个变量作为参数调用 `event_base_new_with_config`。因为并没有对 `cfg` 进行任何的设置，所以得到的是默认配置的 `event_base`。

## 定制 event_base

从以上分析也可以知道，如果要对 `event_base` 进行配置，那么对 `cfg` 变量进行配置即可。现在我们的目光从 `event_base` 结构体转到 `event_config` 结构体。

`event_config` 结构体定义如下：

``` cpp
struct event_config {
    TAILQ_HEAD(event_configq, event_config_entry) entries;
    int n_cpus_hint;
    struct timeval max_dispatch_interval;
    int max_dispatch_callbacks;
    int limit_callbacks_after_prio;
    enum event_method_feature require_features;
    enum event_base_config_flag flags;
};
```

`event_config` 结构体中定义了 7 个变量，对应了 `event_base` 的 5 中配置选项：

1. `entries` - 拒绝使用某个多路 `IO` 复用函数
2. `n_cpus_hint` - 指明 `CPU` 的数量，以便 `libevent` 内部进行 `CPU` 优化
3. `max_dispatch_interval`/`max_dispatch_callbacks`/`limit_callbacks_after_prio` 联合限制 libevent 的事件回调频率
4. `require_features` - 指明多路 `I/O` 复用函数需要具有的特征
5. `flags` - 其他的一些特征标志位

### 拒绝使用某个多路 `IO` 复用函数

第一个成员 `entries`，其结构是一个队列，队列元素的类型就是 `event_config_entry`，可以用来存储一个字符串指针。它对应的设置函数为 `event_config_avoid_method`。

``` cpp
struct event_config_entry {
    TAILQ_ENTRY(event_config_entry) next;
    const char *avoid_method;
};

int event_config_avoid_method(struct event_config *cfg, const char *method) {
    struct event_config_entry *entry = mm_malloc(sizeof(*entry));
    if (entry == NULL) return (-1);
    if ((entry->avoid_method = mm_strdup(method)) == NULL) {
        mm_free(entry);
        return (-1);
    }
    TAILQ_INSERT_TAIL(&cfg->entries, entry, next);
    return (0);
}
```

`libevent` 是跨平台的 `Reactor`，对于事件监听，其内部是使用多路 `IO` 复用函数。比较通用的多路 `IO` 复用函数是 `select` 和 `poll`。而很多平台都提出了自己的高效多路 `IO` 复用函数，比如：`epoll`、`devpoll`、`kqueue`。`libevent` 对于这些多路 `IO` 复用函数都进行包装，供自己使用。`event_config_avoid_method` 函数就是指出，避免使用指定的多路 `IO` 复用函数。其是通过字符串的方式指定的，即参数 `method`。这个字符串将由队列元素 `event_config_entry` 的 `avoid_method` 成员变量存储 (由于是指针，所以更准确来说是指向)。

查看 `libevent` 源码包里的文件，可以发现有诸如 `epoll.c`、`select.c`、`poll.c`、`devpoll.c`、`kqueue.c` 这些文件。打开这些文件就可以发现在文件内容的前面都会定义一个 `struct eventop` 类型变量。该结构体的第一个成员必然是一个字符串。这个字符串就描述了对应的多路 `IO` 复用函数的名称。所以是可以通过名称来禁用某种多路 `IO` 复用函数的。

``` cpp
/* select.c */
const struct eventop selectops = {
    "select",
    select_init,
    select_add,
    select_del,
    select_dispatch,
    select_dealloc,
    0, /* doesn't need reinit. */
    EV_FEATURE_FDS,
    0,
};
```

### 智能调整 CPU 个数

第二个成员变量 `n_cpus_hint`。从名字来看是指明 `CPU` 的数量。是通过函数 `event_config_set_num_cpus_hint` 来设置的。其作用是告诉 `event_config`，系统中有多少个 `CPU`，以便作一些对线程池作一些调整来获取更高的效率。目前，仅仅 `Window` 系统的 `IOCP`(`Windows` 的 `IOCP` 能够根据 `CPU` 的个数智能调整)，该函数的设置才有用。在以后，`libevent` 可能会将之应用于其他系统。

正如其名字中的 `hint`，这仅仅是一个提示。就如同 `C++` 中的 `inline`。`event_base` 实际使用的 `CPU` 个数不一定等于提示的个数。

### 规定多路 `IO` 复用函数需满足的特征

第六个成员变量 `require_features`。从其名称来看是要求的特征。不错，这个变量指定 多路 `IO` 复用函数应该满足哪些特征。所有的特征定义在一个枚举类型中。

``` cpp
enum event_method_feature {
    EV_FEATURE_ET = 0x01, /* 支持边沿触发 */
    EV_FEATURE_O1 = 0x02, /* 添加、删除、或者确定哪个事件激活这些动作的时间复杂度都为 O(1) */
    EV_FEATURE_FDS = 0x04, /* 支持任意的文件描述符，而不能仅仅支持套接字 */
    EV_FEATURE_EARLY_CLOSE = 0x08 /* 支持在不读取接收缓冲区数据时，检测连接关闭 */
};
```

比如 epoll.c 中定义了 epoll 满足的特性：

``` cpp
const struct eventop epollops = {
    "epoll",
    epoll_init,
    epoll_nochangelist_add,
    epoll_nochangelist_del,
    epoll_dispatch,
    epoll_dealloc,
    1, /* need reinit */
    EV_FEATURE_ET|EV_FEATURE_O1|EV_FEATURE_EARLY_CLOSE,
    0
};
```

### 其他配置标志位

`event_config` 的第四个成员 `flags`，代表了一些 `event_base` 对象的特征，其通过标志位来进行表示。

可用的标志位可参见枚举类型 `event_base_config_flag`，该枚举类型定义如下：

``` cpp
enum event_base_config_flag {
    /** 不要为 event_base 分配锁。
    *** 设置这个选项可以为 event_base 节省一点加锁和解锁的时间，但是当多个线程访问 event_base 会变得不安全 */
    EVENT_BASE_FLAG_NOLOCK = 0x01,

    /** 选择多路 IO 复用函数时，不检测 EVENT_* 环境变量。
    *** 使用这个标志要考虑清楚：因为这会使得用户更难调试程序与 libevent 之间的交互 */
    EVENT_BASE_FLAG_IGNORE_ENV = 0x02,

    /** 仅用于 Windows。这使得 libevent 在启动时就启用任何必需的 IOCP 分发逻辑，而不是按需启用。
    *** 如果设置了这个宏，那么 evconn_listener_new 和 bufferevent_socket_new 函数的内部将使用 IOCP */
    EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,

    /** 在执行 event_base_loop 的时候没有 cache 时间。
    *** 该函数的 while 循环会经常取系统时间，如果有 cache 时间，那么就取 cache 的。如果没有的话，就只能通过系统提供的函数来获取系统时间。这将更耗时 */
    EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,

    /** 告知 Libevent，如果决定使用 epoll 这个多路 IO 复用函数，可以安全地使用更快的基于 changelist 的多路 IO 复用函数：epollchangelist
    *** 多路 IO 复用可以在多路 IO 复用函数调用之间，同样的 fd 多次修改其状态的情况下，避免不必要的系统调用
    *** 但是如果传递任何使用 dup() 或者其变体克隆的 fd 给 libevent，epollchangelist 多路 IO 复用函数会触发一个内核 bug，导致不正确的结果
    *** 在不使用 epoll 这个多路 IO 复用函数的情况下，这个标志是没有效果的。也可以通过设置 EVENT_EPOLL_USE_CHANGELIST 环境变量来打开 epollchangelist 选项 */
    EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,

    /** 通常，libevent 使用我们拥有的最快的单调计时器实现其时间和超时代码。
    *** 如果设置了此标志，libevent 将使用一个存在的且效率较低的更精确的计时器。 */
    EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
```

## 根据配置创建 event_base

将以上我们定制好的 `event_config` 类型的参数 `cfg` 传入到 `event_base_new_with_config` 中，即可根据 `cfg` 中的配置来创建符合条件的 `event_base`，`event_base_new_with_config` 中部分关键代码如下：

``` cpp
struct event_base * event_base_new_with_config(const struct event_config *cfg) {
    int i;
    struct event_base *base;
    int should_check_environment;

#ifndef EVENT__DISABLE_DEBUG_MODE
    event_debug_mode_too_late = 1;
#endif

    if ((base = mm_calloc(1, sizeof(struct event_base))) == NULL) {
        event_warn("%s: calloc", __func__);
        return NULL;
    }

    if (cfg) base->flags = cfg->flags;

    should_check_environment = !(cfg && (cfg->flags & EVENT_BASE_FLAG_IGNORE_ENV));

    ......

    base->evbase = NULL;

    if (cfg) {
        memcpy(&base->max_dispatch_time, &cfg->max_dispatch_interval, sizeof(struct timeval));
        base->limit_callbacks_after_prio = cfg->limit_callbacks_after_prio;
    } else {
        base->max_dispatch_time.tv_sec = -1;
        base->limit_callbacks_after_prio = 1;
    }
    if (cfg && cfg->max_dispatch_callbacks >= 0) base->max_dispatch_callbacks = cfg->max_dispatch_callbacks;
    else base->max_dispatch_callbacks = INT_MAX;

    if (base->max_dispatch_callbacks == INT_MAX && base->max_dispatch_time.tv_sec == -1)
        base->limit_callbacks_after_prio = INT_MAX;

    for (i = 0; eventops[i] && !base->evbase; i++) {
        if (cfg != NULL) {
            /* determine if this backend should be avoided */
            if (event_config_is_avoided_method(cfg, eventops[i]->name)) continue;
            if ((eventops[i]->features & cfg->require_features) != cfg->require_features) continue;
        }

        /* also obey the environment variables */
        if (should_check_environment && event_is_method_disabled(eventops[i]->name)) continue;

        base->evsel = eventops[i];
        base->evbase = base->evsel->init(base);
    }

    if (base->evbase == NULL) {
        event_warnx("%s: no event mechanism available", __func__);
        base->evsel = NULL;
        event_base_free(base);
        return NULL;
    }

    if (evutil_getenv_("EVENT_SHOW_METHOD")) event_msgx("libevent using: %s", base->evsel->name);

    ......

    return (base);
}
```

从代码中可以看出，`event_base_new_with_config` 做的最重要的一个工作就是选择一个合适的多路 `IO` 复用函数。

### 多路 IO 复用函数的选择

多路 `IO` 复用函数方法也比较简单：

`libevent` 根据宏定义判断当前的 `OS` 环境是否有某个多路 `IO` 复用函数。如果有，那么就把与之对应的 `struct eventop` 结构体指针放到一个全局数组 `eventops` 中。有了这个数组，就只需遍历这个数组，并将其中某个符合条件的元素赋值给 `event_base.evsel` 变量即可。

`eventops` 数组和 `event_base.evsel` 都是一个结构体指针，结构体类型为 `struct eventop`：

``` cpp
struct eventop {
    const char *name;
    void *(*init)(struct event_base *);
    int (*add)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
    int (*del)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
    int (*dispatch)(struct event_base *, struct timeval *);
    void (*dealloc)(struct event_base *);
    int need_reinit;
    enum event_method_feature features;
    size_t fdinfo_len;
};
```

其中各参数的作用为（`libevent` 中 `backend`，直译为“后端”，在 `libevent` 中与多路 `IO` 复用函数意义相同，下文中会混用两个概念）：

1. `name`：多路 `IO` 复用函数名称
2. `init`：用于设置 `event_base` 以使用此后端的函数。它应该创建一个新结构，其中包含运行后端并返回它所需的所有信息。返回的指针将由 `event_init` 存储到 `event_base.evbase` 字段中。失败时，此函数应返回 `NULL`
3. `add`：启用对给定 `fd` 或信号的读取 / 写入。 `events` 将是我们要启用的事件：`EV_READ`，`EV_WRITE`，`EV_SIGNAL` 和 `EV_ET` 中的一个或多个。 `old` 是先前在此 `fd` 上启用了那些事件。 `fdinfo` 将是 `evmap` 与 `fd` 相关联的结构；它的大小由下面的 `fdinfo` 字段定义。首次添加 `fd` 时，它将设置为 `0`。该函数成功返回 `0`，错误返回 `-1`。
4. `del`：与添加相对应，禁用我们要排除的事件。
5. `dispatch`：函数实现事件循环的核心。它必须查看添加哪些 `events` 已经处于准备状态，并为每个活动事件调用 `event_active`（通常通过 `event_io_active` 等）。成功返回 `0`，错误返回 `-1`。
6. `dealloc`：用于从 `event_base` 中清除并释放数据的函数。
7. `need_reinit`：标志：设置我们是否需要在 `fork` 后重新初始化事件库。
8. `features`：按位表示此多路 `IO` 复用函数可以提供的受支持的 `event_method_features` 特性。
9. `fdinfo_len`：我们应该为具有一个或多个活动事件的每个 `fd` 记录的额外信息的长度。此信息记录为每个 `fd` 的 `evmap` 条目的一部分，并作为参数传递给上面的 `add` 和 `del` 函数。

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [libevent-src-analysis/libevent 源码分析. md at master · KelvinYin/libevent-src-analysis](https://github.com/KelvinYin/libevent-src-analysis/blob/master/libevent%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
