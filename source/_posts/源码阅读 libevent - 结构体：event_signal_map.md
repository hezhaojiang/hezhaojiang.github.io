---
title: 源码阅读 libevent - 结构体：event_signal_map
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: a24039d7
date: 2020-10-31 06:03:13
---
在之前的文章中提到过：`libevent` 中的哈希表只会用于 `Windows` 系统，像遵循 `POSIX` 标准的 `OS` 是不会用到哈希表的。那么在遵循 `POSIX` 标准的 `OS` 中 event_io_map 是怎么实现的呢？答案就在下面的宏定义中：

``` c++
#define event_io_map event_signal_map
```

从该宏定义中可以看出，此时 `event_io_map` 被定义为了 `event_signal_map` 结构体。本文就分析一下 `event_signal_map` 结构体。

<!--more-->

## event_signal_map 的定义

``` c++
struct event_signal_map {
    void **entries; /* evmap_io * 或 evmap_signal * 型数组 */
    int nentries; /* 数组有效长度 */
};
```

`event_signal_map` 结构体中仅有两个元素，其意义如下：

1. `entries`：一个二级指针，其实是一个数组，数组中元素为指针类型（`evmap_io*` 型或 `evmap_signal*` 型）
2. `nentries`：表明 `entries` 数组的有效长度，以防止访问溢出。

在 {% post_link "源码阅读 libevent - event_io_map" %} 文章中已经分析过 `evmap_io` 结构体，本文侧重分析 `evmap_signal` 结构体。

### evmap_signal

``` c++
struct evmap_signal {
    struct event_dlist events; /* LIST_HEAD (event_dlist, event); */
};
```

`evmap_signal` 结构体也极其简单，其中仅有一个元素，是一个双向链表的表头。

## event_signal_map 的结构

从上述定义分析就可看出 `event_signal_map` 的结构较为简单：

``` node
event_signal_map        Array of struct evmap_signal*       struct event             struct event
+--------------+        +----------------------------+      +-------------+          +-------------+
|   entries    |------->| event_dlist (DList Header) |----->|   DList A   |-- ... -->|   DList N   |-----> NULL
+--------------+        +----------------------------+      +-------------+          +-------------+
|   nentries   |        | event_dlist (DList Header) |-----> NULL
+--------------+        +----------------------------+
                        | event_dlist (DList Header) |-----> NULL
                        +----------------------------+
                                    ......
                        +----------------------------+      +-------------+
                        | event_dlist (DList Header) |----->|   DList X   |-----> NULL
                        +----------------------------+      +-------------+
```

## 初始化

``` c++
void evmap_signal_initmap_(struct event_signal_map *ctx) {
    ctx->nentries = 0;
    ctx->entries = NULL;
}
```

Emmm… 不知道该说点什么...

## 添加

``` c++
int evmap_signal_add_(struct event_base *base, int sig, struct event *ev) {
    const struct eventop *evsel = base->evsigsel;
    struct event_signal_map *map = &base->sigmap;
    struct evmap_signal *ctx = NULL;

    if (sig < 0 || sig>= NSIG) return (-1);
    if (sig>= map->nentries)
        if (evmap_make_space(map, sig, sizeof(struct evmap_signal *)) == -1) return (-1);
    GET_SIGNAL_SLOT_AND_CTOR(ctx, map, sig, evmap_signal, evmap_signal_init, base->evsigsel->fdinfo_len);
    if (LIST_EMPTY(&ctx->events))
        if (evsel->add(base, ev->ev_fd, 0, EV_SIGNAL, NULL) == -1) return (-1);
    LIST_INSERT_HEAD(&ctx->events, ev, ev_signal_next);
    return (1);
}
```

以上代码可分为 4 个步骤：

1. evmap_make_space
2. GET_SIGNAL_SLOT_AND_CTOR
3. event_base.evsel->add
4. LIST_INSERT_HEAD

其中步骤 3 和步骤 4 在 {% post_link "源码阅读 libevent - event_io_map" %} 文章中已讲解过，下面分析前两个步骤：

### evmap_make_space

``` c++
static int evmap_make_space(struct event_signal_map *map, int slot, int msize) {
    if (map->nentries <= slot) {
        int nentries = map->nentries ? map->nentries : 32;
        void **tmp;

        if (slot> INT_MAX / 2) return (-1);
        while (nentries <= slot) nentries <<= 1;
        if (nentries> INT_MAX / msize) return (-1);

        tmp = (void **)mm_realloc(map->entries, nentries * msize);
        if (tmp == NULL) return (-1);
        memset(&tmp[map->nentries], 0, (nentries - map->nentries) * msize);

        map->nentries = nentries;
        map->entries = tmp;
    }
    return (0);
}
```

`evmap_make_space` 函数会在数组长度不足时对数组进行拓展：

1. 如果数组没有初始化，则初始化数组长度为 `32`
2. 如果数组长度不足，则将数组长度乘 `2`，直至满足要求
3. `realloc` 数组空间，并将新开辟出来的空间清 `0`
4. 保存新数组的地址信息和长度信息

### GET_SIGNAL_SLOT_AND_CTOR

``` c++
GET_SIGNAL_SLOT_AND_CTOR(ctx, map, sig, evmap_signal, evmap_signal_init, base->evsigsel->fdinfo_len);
/* 以上宏定义展开后结果为如下所示 */
do {
    if ((map)->entries[sig] == NULL) {
        (map)->entries[sig] = mm_calloc(1,sizeof(struct evmap_signal)+base->evsigsel->fdinfo_len);
        if (EVUTIL_UNLIKELY((map)->entries[sig] == NULL)) return (-1);
        (evmap_signal_init)((struct evmap_signal *)(map)->entries[sig]);
    }
    (ctx) = (struct evmap_signal *)((map)->entries[sig]);
} while (0)
```

`GET_SIGNAL_SLOT_AND_CTOR` 宏找到当前 `sig` 对应的数组元素：

- 如果该元素为空，则新建一个双向链表的表头，存入链表，并将该表头地址赋值给 ctx 变量
- 如果该元素非空，则其指向的是一个双向链表的表头，将该元素赋值给 ctx 变量

## 删除

``` c++
int evmap_signal_del_(struct event_base *base, int sig, struct event *ev) {
    const struct eventop *evsel = base->evsigsel;
    struct event_signal_map *map = &base->sigmap;
    struct evmap_signal *ctx;

    if (sig < 0 || sig>= map->nentries) return (-1);
    GET_SIGNAL_SLOT(ctx, map, sig, evmap_signal);
    LIST_REMOVE(ev, ev_signal_next);

    if (LIST_FIRST(&ctx->events) == NULL)
        if (evsel->del(base, ev->ev_fd, 0, EV_SIGNAL, NULL) == -1) return (-1);
    return (1);
}
```

### GET_SIGNAL_SLOT

``` c++
GET_SIGNAL_SLOT(ctx, map, sig, evmap_signal);
/* 以上宏定义展开后结果为如下所示 */
(ctx) = (struct evmap_signal *)((map)->entries[sig])
```

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [libevent-src-analysis/libevent 源码分析. md at master · KelvinYin/libevent-src-analysis](https://github.com/KelvinYin/libevent-src-analysis/blob/master/libevent%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
