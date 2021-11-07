---
title: 源码阅读 libevent - 结构体：event_io_map
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 517d2de7
date: 2020-10-29 16:30:24
---
之前的文章 {% post_link "源码阅读 libevent - 数据结构：哈希表" %} 分析了 `libevent` 中哈希表的实现，本文我们分析 `libevent` 中的利用哈希表实现的结构体：`event_io_map`。

> 本系列大部分文章介绍 `linux` 系统下 `libevent` 的源码，但 `event_io_map` 在 `linux` 环境下与 `event_signal_map` 一致，而且其在 `Windows` 下的实现更值得学习。

<!--more-->

## event_io_map 的定义

`libevent` 中的哈希表只会用于 `Windows` 系统，像遵循 `POSIX` 标准的 `OS` 是不会用到哈希表的。从下面的定义可以看到这一点。

``` c++
#ifdef _WIN32
#define EVMAP_USE_HT
#endif

#ifdef EVMAP_USE_HT
#define HT_NO_CACHE_HASH_VALUES
#include "ht-internal.h"
struct event_map_entry;
HT_HEAD(event_io_map, event_map_entry);
#else
#define event_io_map event_signal_map
#endif

struct event_signal_map {
    void **entries; /* An array of evmap_io * or of evmap_signal *; empty entries are set to NULL. */
    int nentries;   /* The number of entries available in entries */
};
```

因为在 `Windows` 系统里面，文件描述符是一个比较大的值，不适合放到 `event_signal_map` 结构中。而通过哈希 (模上一个小的值)，就可以变得比较小，这样就可以放到哈希表的数组中了。而遵循 `POSIX` 标准的文件描述符是从 `0` 开始递增的，一般都不会太大，适用于 `event_signal_map`，此时 `event_io_map` 被定义为了 `event_signal_map` 结构体。

本文分析的是 `event_io_map` 的哈希实现方式。

## event_io_map 结构

`event_io_map` 结构几个重要结构体如下：

``` c++
LIST_HEAD (event_dlist, event);
struct evmap_io {
    struct event_dlist events; /* LIST_HEAD (event_dlist, event); */
    ev_uint16_t nread;
    ev_uint16_t nwrite;
    ev_uint16_t nclose;
};

struct event_map_entry {
    HT_ENTRY(event_map_entry) map_node;
    evutil_socket_t fd;
    union { /* This is a union in case we need to make more things that can be in the hashtable. */
        struct evmap_io evmap_io;
    } ent;
};
HT_HEAD(event_io_map, event_map_entry);
```

从之前的文章（参见：{% post_link "源码阅读 libevent - 数据结构：哈希表" %}）分析可知：

- 哈希表的表头为 `event_io_map` 类型
- 哈希表中的桶为 `event_map_entry*` 类型的数组
- 哈希表中的元素为 `event_map_entry` 类型
- `event_map_entry` 类型中的 `Hash Key` 为 `evutil_socket_t fd`
- `event_map_entry` 类型中的 `Hash Value` 为 `struct evmap_io evmap_io`
- `struct evmap_io` 类型中有一个双向链表（参见：{% post_link "源码阅读 libevent - 数据结构：双向链表" %}）

## 初始化

`event_base_new_with_config` 函数中会调用 `evmap_io_initmap_` 函数对每个 `event_base` 中的 `event_io_map` 进行初始化：

``` c++
/** file: event.c
*** function: event_base_new_with_config */
evmap_io_initmap_(&base->io);

void evmap_io_initmap_(struct event_io_map *ctx) {
    HT_INIT(event_io_map, ctx);
}
```

## 添加

``` c++
int evmap_io_add_(struct event_base *base, evutil_socket_t fd, struct event *ev) {
    const struct eventop *evsel = base->evsel;
    struct event_io_map *io = &base->io;
    struct evmap_io *ctx = NULL;
    int nread, nwrite, nclose, retval = 0;
    short res = 0, old = 0;
    struct event *old_ev;
    EVUTIL_ASSERT(fd == ev->ev_fd);

    if (fd < 0) return 0;
    GET_IO_SLOT_AND_CTOR(ctx, io, fd, evmap_io, evmap_io_init, evsel->fdinfo_len);

    nread = ctx->nread;
    nwrite = ctx->nwrite;
    nclose = ctx->nclose;

    if (nread) old |= EV_READ;
    if (nwrite) old |= EV_WRITE;
    if (nclose) old |= EV_CLOSED;
    if (ev->ev_events & EV_READ) if (++nread == 1) res |= EV_READ;
    if (ev->ev_events & EV_WRITE) if (++nwrite == 1) res |= EV_WRITE;
    if (ev->ev_events & EV_CLOSED) if (++nclose == 1) res |= EV_CLOSED;

    ......

    if (res) {
        void *extra = ((char*)ctx) + sizeof(struct evmap_io);
        if (evsel->add(base, ev->ev_fd, old, (ev->ev_events & EV_ET) | res, extra) == -1) return (-1);
        retval = 1;
    }

    ctx->nread = (ev_uint16_t) nread;
    ctx->nwrite = (ev_uint16_t) nwrite;
    ctx->nclose = (ev_uint16_t) nclose;
    LIST_INSERT_HEAD(&ctx->events, ev, ev_io_next);

    return (retval);
}
```

### GET_IO_SLOT_AND_CTOR

``` c++
GET_IO_SLOT_AND_CTOR(ctx, io, fd, evmap_io, evmap_io_init, evsel->fdinfo_len);
/* 以上宏定义展开后结果为如下所示 */
do {
    struct event_map_entry key_, *ent_;
    key_.fd = fd;
    HT_FIND_OR_INSERT_(event_io_map, map_node, hashsocket, io, event_map_entry, &key_, ptr,
        {
            ent_ = *ptr;
        },
        {
            ent_ = event_mm_calloc_((1), (sizeof(struct event_map_entry) + evsel->fdinfo_len));
            if (EVUTIL_UNLIKELY(ent_ == NULL)) return (-1);
            ent_->fd = fd; (evmap_io_init)(&ent_->ent.evmap_io);
            HT_FOI_INSERT_(map_node, io, &key_, ent_, ptr)
        });
    (ctx) = &ent_->ent.evmap_io;
}
while (0);
```

其中 `HT_FIND_OR_INSERT_` 宏和 `HT_FOI_INSERT_` 宏共同实现了 `Find or Insert` 功能：

- 如果被查找的 `fd` 在哈希表中，则 `ctx` 被赋值为该 `fd` 对应的 `evmap_io` 双向链表头所在地址
- 如果被查找的 `fd` 没有在哈希表中，则申请双向链表头，初始化后插入到哈希表中，并返回该双向链表头所在地址

双向链表的初始化函数如下：

``` c++
static void evmap_io_init(struct evmap_io *entry) {
    LIST_INIT(&entry->events);
    entry->nread = 0;
    entry->nwrite = 0;
    entry->nclose = 0;
}
```

### event_base.evsel->add

`event_base.evsel->add` 其实是个函数指针（参见：{% post_link "源码阅读 libevent - 创建 event_base" %}），其根据 `event_base` 选择的不同后端来调用不同的函数来执行事件添加操作，以 `select` 后端为例：

``` c++
static int select_add(struct event_base *base, int fd, short old, short events, void *p) {
    struct selectop *sop = base->evbase; /* event_base.evbase 是 init 返回的结构体指针 */
    EVUTIL_ASSERT((events & EV_SIGNAL) == 0);

    if (sop->event_fds < fd) {
        int fdsz = sop->event_fdsz;
        if (fdsz < (int)sizeof(fd_mask)) fdsz = (int)sizeof(fd_mask);
        while (fdsz < (int) SELECT_ALLOC_SIZE(fd + 1)) fdsz *= 2;

        if (fdsz != sop->event_fdsz)
            if (select_resize(sop, fdsz)) return (-1);
        sop->event_fds = fd;
    }
    if (events & EV_READ) FD_SET(fd, sop->event_readset_in);
    if (events & EV_WRITE) FD_SET(fd, sop->event_writeset_in);

    return (0);
}
```

从上述代码来看，`event_base.evsel->add` 本质上是调用了对应后端的函数集中对应的函数来实现事件的添加操作，如 `select` 就是调用 `FD_SET` 来实现事件的添加操作。

### LIST_INSERT_HEAD

在 `GET_IO_SLOT_AND_CTOR` 宏返回了双向链表头所在地址后，通过 `LIST_INSERT_HEAD` 宏将新事件 `ev` 插入到双向链表中。

> 有关 `LIST_INSERT_HEAD` 的实现，参见：post_link "源码阅读 libevent - 数据结构：双向链表" %}

## 删除

``` c++
int evmap_io_del_(struct event_base *base, evutil_socket_t fd, struct event *ev) {
    const struct eventop *evsel = base->evsel;
    struct event_io_map *io = &base->io;
    struct evmap_io *ctx;
    int nread, nwrite, nclose, retval = 0;
    short res = 0, old = 0;

    if (fd < 0) return 0;
    EVUTIL_ASSERT(fd == ev->ev_fd);
    GET_IO_SLOT(ctx, io, fd, evmap_io);

    nread = ctx->nread;
    nwrite = ctx->nwrite;
    nclose = ctx->nclose;

    if (nread) old |= EV_READ;
    if (nwrite) old |= EV_WRITE;
    if (nclose) old |= EV_CLOSED;

    if (ev->ev_events & EV_READ) {
        if (--nread == 0) res |= EV_READ;
        EVUTIL_ASSERT(nread>= 0);
    }
    if (ev->ev_events & EV_WRITE) {
        if (--nwrite == 0) res |= EV_WRITE;
        EVUTIL_ASSERT(nwrite>= 0);
    }
    if (ev->ev_events & EV_CLOSED) {
        if (--nclose == 0) res |= EV_CLOSED;
        EVUTIL_ASSERT(nclose>= 0);
    }

    if (res) {
        void *extra = ((char*)ctx) + sizeof(struct evmap_io);
        if (evsel->del(base, ev->ev_fd, old, (ev->ev_events & EV_ET) | res, extra) == -1) retval = -1;}
        else retval = 1;
    }

    ctx->nread = nread;
    ctx->nwrite = nwrite;
    ctx->nclose = nclose;
    LIST_REMOVE(ev, ev_io_next);

    return (retval);
}
```

### GET_IO_SLOT

``` c++
GET_IO_SLOT(ctx, io, fd, evmap_io);
/* 以上宏定义展开后结果为如下所示 */
do {
    struct event_map_entry key_, *ent_;
    key_.fd = fd;
    ent_ = HT_FIND(event_io_map, io, &key_);
    (ctx) = ent_ ? &ent_->ent.evmap_io : NULL;
} while (0);
```

该宏定义与直接调用 `HT_FIND` 的区别是：`HT_FIND` 宏 `find` 的结果是 哈希表中的元素地址，这里的元素包括哈希表的 `Key/Value/Next`，而 `GET_IO_SLOT` 是调用 `HT_FIND` 查找到需要的元素地址后，仅仅是将需要用的到 `Value` 的地址赋值给 `ctx` 变量，这里需要用到的 `Value` 是一个 `struct evmap_io*` 类型的指针，该类型在上文中已经描述，此处不再重复，需要注意的是其中保存有一个双向链表的表头。

### event_base.evsel->del

``` c++
static int select_del(struct event_base *base, int fd, short old, short events, void *p) {
    struct selectop *sop = base->evbase;
    EVUTIL_ASSERT((events & EV_SIGNAL) == 0);

    if (sop->event_fds < fd) return (0);
    if (events & EV_READ) FD_CLR(fd, sop->event_readset_in);
    if (events & EV_WRITE) FD_CLR(fd, sop->event_writeset_in);

    return (0);
}
```

和 `event_base.evsel->add` 一样，`event_base.evsel->del` 也是调用了对应后端的函数集中对应的函数来实现事件的删除操作，如 `select` 就是调用 `FD_CLR` 来实现事件的删除操作。

### LIST_REMOVE

将 `ev` 事件从链表中删除。

> 有关 `LIST_REMOVE` 的实现，参见：post_link "源码阅读 libevent - 数据结构：双向链表" %}

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [libevent-src-analysis/libevent 源码分析. md at master · KelvinYin/libevent-src-analysis](https://github.com/KelvinYin/libevent-src-analysis/blob/master/libevent%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
