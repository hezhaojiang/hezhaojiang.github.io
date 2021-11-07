---
title: 源码阅读 libevent - 结构体：event
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: afa6fb2f
date: 2020-11-01 13:54:26
---
在 {% post_link "源码阅读 libevent - event_io_map" %} 和 {% post_link "源码阅读 libevent - event_signal_map" %} 中，无论是哈希表还是普通数组，都是将一个 fd 或者 sig 映射到了一个双向链表的表头上：

``` c++
LIST_HEAD (event_dlist, event);
/* 以上宏定义展开后结果为如下所示 */
struct event_dlist {
    struct event *lh_first;  /* first element */
}
```

该双向量表的表头为结构体 `struct event_dlist`，链表中元素的结构体为 `struct event`，其中每个 `struct event` 链表元素代表着一个事件，本文主要分析这个事件结构体：`struct event`。

<!--more-->

## struct event 的定义

``` c++
struct event {
    struct event_callback ev_evcallback;

    union { /* for managing timeouts */
        TAILQ_ENTRY(event) ev_next_with_common_timeout;
        int min_heap_idx;
    } ev_timeout_pos;

    evutil_socket_t ev_fd; /* 对于 `I/O` 事件是文件描述符，对于 `signal` 事件是信号值 */
    struct event_base *ev_base; /* 与 event 对应的 event_base */

    union {
        struct { /* used for io events */
            LIST_ENTRY (event) ev_io_next;
            struct timeval ev_timeout;
        } ev_io;
        struct { /* used by signal events */
            LIST_ENTRY (event) ev_signal_next;
            short ev_ncalls;
            short *ev_pncalls; /* Allows deletes in callback */
        } ev_signal;
    } ev_;

    short ev_events; /* 关注的事件类型 超时、读、写、永久 */
    short ev_res;    /* result passed to event callback 事件激活的类型 */
    struct timeval ev_timeout; /* 超时时间，存储的是一个绝对超时值（从 1970 年 1 月 1 日开始） */
};
```

从结构体定义中可以发现，其中有几个我们之前文章提过的结构，如：`TAILQ_ENTRY`、`LIST_ENTRY`，这代表同一个 event 结构体可能会出现在多个不同的链表或队列中。

下面我们分块了解下每个结构体成员：

### ev_evcallback

`ev_evcallback` 的定义为：`struct event_callback ev_evcallback;`，其中 `struct event_callback` 定义如下：

``` c++
struct event_callback {
    TAILQ_ENTRY(event_callback) evcb_active_next;
    short evcb_flags;
    ev_uint8_t evcb_pri;    /* smaller numbers are higher priority */
    ev_uint8_t evcb_closure;
    /* allows us to adopt for different types of events */
    union {
        void (*evcb_callback)(evutil_socket_t, short, void *);
        void (*evcb_selfcb)(struct event_callback *, void *);
        void (*evcb_evfinalize)(struct event *, void *);
        void (*evcb_cbfinalize)(struct event_callback *, void *);
    } evcb_cb_union;
    void *evcb_arg;
};
```

1. `evcb_active_next`：尾队列的成员，保存其在尾队列中的位置等信息（注意，该尾队列的元素类型为 `struct event_callback`，而非 `struct event` ）
2. `evcb_flags`：反映 `event` 目前的状态，是处于超时队列、已添加队列、激活队列、信号队列等
    - 描述 `event` 的状态，由 `libevent` 内部设置。比如说 `event` 被初始化，那么 `flags` 就会设置为 `EVLIST_INIT`，有如下几种状态：

    ``` c++
    #define EVLIST_TIMEOUT      0x01    // event 在 time min_heap 中
    #define EVLIST_INSERTED     0x02    // event 在已注册事件链表中
    #define EVLIST_SIGNAL       0x04    // event 属于信号队列
    #define EVLIST_ACTIVE       0x08    // event 在激活链表中
    #define EVLIST_INTERNAL     0x10    // event 为内部使用
    #define EVLIST_ACTIVE_LATER 0x20    // event 在下一次激活链表中
    #define EVLIST_FINALIZING   0x40    // event 已经终止
    #define EVLIST_INIT         0x80    // event 已被初始化，但是哪儿都不在

    #define EVLIST_ALL          0xff    // 包含所有事件状态，用于判断合法性
    ```

3. `evcb_pri`：回调优先级，数字越小优先级越高，实际上就是 `event` 激活时在激活队列中的索引值
4. `evcb_closure`：描述 event 在激活时的处理方式。
    - 对于永久事件来说就需要重新添加到定时器中并调用回调函数，而对于一般的事件来说则是直接调用回调函数。`ev_closure` 可以设置为以下几种：

    ``` c++
    #define EV_CLOSURE_EVENT                0   // 常规事件，使用 evcb_callback 回调
    #define EV_CLOSURE_EVENT_SIGNAL         1   // 信号事件；使用 evcb_callback 回调
    #define EV_CLOSURE_EVENT_PERSIST        2   // 永久性非信号事件；使用 evcb_callback 回调
    #define EV_CLOSURE_CB_SELF              3   // 简单回调，使用 evcb_selfcb 回调
    #define EV_CLOSURE_CB_FINALIZE          4   // 结束的回调，使用 evcb_cbfinalize 回调
    #define EV_CLOSURE_EVENT_FINALIZE       5   // 结束事件回调，使用 evcb_evfinalize 回调
    #define EV_CLOSURE_EVENT_FINALIZE_FREE  6   // 结束事件之后应该释放，使用 evcb_evfinalize 回调
    ```

5. `evcb_cb_union`：回调函数联合体，其中根据不同的事件类型决定其回调函数的类型
6. `evcb_arg`：传给回调函数的参数

### ev_timeout_pos

``` c++
union { /* for managing timeouts */
    TAILQ_ENTRY(event) ev_next_with_common_timeout;
    int min_heap_idx;
} ev_timeout_pos;
```

`libevent` 有两种超时管理机制：

- `min_heap` 机制：使用 `min_heap_idx`，
- `min_heap`+`common_timeout` 机制：使用 `ev_next_with_common_timeout`

上述两种机制只会使用其中一个，使用联合体存储更加节约空间。

### ev_base

`ev_base` 是一个指向 `event_base` 的指针，用于描述该 `event` 所属的 `event_base`。

在 `libevent` 中，每一个 `event_base` 都定义了以下几种事件集合：已添加事件队列 `eventqueue`、已激活事件队列 `activequeues`、定时器 `min_heap`、公用超时事件队列 `common_timeout_queues`、`io` 事件集合以及 `signal` 事件集合，如下所示：

``` c++
struct event_base {
    ......
    /** An array of nactivequeues queues for active events (ones that
        * have triggered, and whose callbacks need to be called).  Low
        * priority numbers are more important, and stall higher ones.
        */
    struct event_list *activequeues;   // 激活的事件队列
    /** An array of common_timeout_list* for all of the common timeout
        * values we know. */
    struct common_timeout_list **common_timeout_queues; // 公用超时事件队列
    /** Mapping from file descriptors to enabled (added) events */
    struct event_io_map io; //io 事件集合
    /** Mapping from signal numbers to enabled (added) events. */
    struct event_signal_map sigmap; //signal 事件集合
    /** All events that have been enabled (added) in this event_base */
    struct event_list eventqueue; // 已添加事件队列
    /** Priority queue of events with timeouts. */
    struct min_heap timeheap; // 定时器
    ......
};
```

- 激活事件队列：所有感兴趣事件发生或者已经超时的 `event` 的集合
- 公用超时事件队列：具有相同超时时长的 `event` 的集合
- `io` 事件集合：所有已添加的读写 `event` 的集合
- `signal` 事件集合：所有已添加的信号 `event` 的集合
- 已添加事件队列：所有调用了 `event_add` 进行添加的 `event` 的集合
- 定时器：所有添加了并设置超时时间的 `event` 集合

任何一个 `event`，一旦通过 `event_add` 进行了添加，那么它就会存在于以上几个集合中的一个或多个中。

### ev_

``` c++
union {
    struct { /* used for io events */
        LIST_ENTRY (event) ev_io_next;
        struct timeval ev_timeout;
    } ev_io;
    struct { /* used by signal events */
        LIST_ENTRY (event) ev_signal_next;
        short ev_ncalls;
        short *ev_pncalls; /* Allows deletes in callback */
    } ev_signal;
} ev_;
```

这个联合体中有两个结构体，分别对应了 `IO` 事件和 `Signal` 事件，由于同一个 `fd` 二者不会同时触发，故可用联合体保存两个事件的信息：

- `ev_io`：保存 `IO` 事件信息
    1. `ev_io_next`：该 `event` 在 `event_io_map` 中的双向链表中的位置
    2. `ev_timeout`：如果 `event` 是永久事件，那么该变量就存储设置的超时时长，这是一个相对超时值
- `ev_signal`：保存 `Signal` 事件信息
    1. `ev_signal_next`：该 `event` 在 `event_signal_map` 中的双向链表中的位置
    2. `ev_ncalls`：当 `signal` 事件激活时，调用回调函数的次数
    3. `ev_pncalls`：指向 `ev_ncalls`

> 为什么同一个 `fd` 不会同时触发  `IO` 事件和 `Signal` 事件？参见：{% post_link "源码阅读 libevent - 信号事件处理" %}

### 其他参数

`ev_events`：`event` 感兴趣的事件类型，可以定义为以下组合：

``` c++
#define EV_TIMEOUT  0x01    // 超时事件
/** Wait for a socket or FD to become readable */
#define EV_READ     0x02    // 读事件
/** Wait for a socket or FD to become writeable */
#define EV_WRITE    0x04    // 写事件
/** Wait for a POSIX signal to be raised*/
#define EV_SIGNAL   0x08    // 信号事件
#define EV_PERSIST  0x10    // 永久事件，激活执行后会重新加到队列中等待下一次激活，否则激活执行后会自动移除
/** Select edge-triggered behavior, if supported by the backend. */
#define EV_ET       0x20    // 边沿触发，一般需要后端方法支持
#define EV_FINALIZE 0x40    // 终止事件，如果设置这个选项，则 event_del 不会阻塞，需要使用 event_finalize 或者 event_free_finalize 以保证多线程安全 */
#define EV_CLOSED   0x80    // 关闭事件，可以使用这个选项来检测链接是否关闭，而不需要读取此链接所有未决数据；
```

`ev_timeout`：与前面的 `ev_io` 中的 `ev_timeout` 不同，`ev_io` 中的 `ev_timeout` 保存的是相对超时时长，而这里的 `ev_timeout` 保存的是绝对超时时间。比如说现在 `8:00`，设置 `event` 的超时时长为 `3` 分钟中，那么这个 `3` 分钟实际上是相对于现在来说的 `3` 分钟，也就是相对超时时长，而 `event` 最终会在 `8:03` 时超时，这个 `8:03` 就是绝对超时时间。`libevent` 中绝对超时时间是相对于 `1970` 年 `1` 月 `1` 日 `0` 时来说的。

`ev_res`：`event` 的激活类型，由 `libevent` 内部设置。比如说 `event` 可以设置为 `EV_READ|EV_WRITE` 来监听读和写事件，如果最终 `event` 被读事件激活，那么 `ev_res` 就是 `EV_READ`。

## event 的生命周期

前面提到的中文翻译的文档里，有一张图完美的诠释了 `event` 的生命周期，带箭头实线表示函数调用，实线矩形表示 `event` 的状态，虚线矩形表示用来检测事件状态的函数：

![event 的生命周期](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201128204050.png)

由于 `event` 结构体是 `libevent` 事件处理的核心，涉及问题较多，本文先对图中上半部分进行解析。

## 创建 event

通过前面 `event` 的结构可以知道，`event` 结构体成员分为两类：一类是 `event` 本身的属性，另一类则是用于描述 `event` 在某些 `event` 集合中的位置。前者既然是属性，那么就应当在 `event` 创建时就制定好，而后者则应该是当 `event` 被添加到 `event_base` 之后才进行设置的。那么现在就来看看 `event` 的创建。

``` c++
struct event *
event_new(struct event_base *base, evutil_socket_t fd, short events, void (*cb)(evutil_socket_t, short, void *), void *arg)
{
    struct event *ev;
    ev = mm_malloc(sizeof(struct event));
    if (ev == NULL) return (NULL);
    if (event_assign(ev, base, fd, events, cb, arg) < 0) {
        mm_free(ev);
        return (NULL);
    }

    return (ev);
}
```

主要做了 `2` 个事情：分配内存、初始化结构。

首先通过 `mm_malloc()` 先从堆上分配 `struct event` 的空间，为了监听这个事件，要保证这个事件一直在整个事件循环的过程中是存在的，因此只能从堆上分配。

分配内存后的 `event` 通过 `event_assign()` 根据参数对其各个字段初始化，`event_assign()` 定义如下：

``` c++
int event_assign(struct event *ev, struct event_base *base, evutil_socket_t fd, short events,
                 void (*callback)(evutil_socket_t, short, void *), void *arg) {
    if (!base) base = current_base;
    if (arg == &event_self_cbarg_ptr_) arg = ev;

    if (!(events & EV_SIGNAL)) event_debug_assert_socket_nonblocking_(fd);
    event_debug_assert_not_added_(ev);

    ev->ev_base = base;
    ev->ev_callback = callback;
    ev->ev_arg = arg;
    ev->ev_fd = fd;
    ev->ev_events = events;
    ev->ev_res = 0;
    ev->ev_flags = EVLIST_INIT;
    ev->ev_ncalls = 0;
    ev->ev_pncalls = NULL;

    if (events & EV_SIGNAL) {
        if ((events & (EV_READ|EV_WRITE|EV_CLOSED)) != 0) {
            event_warnx("%s: EV_SIGNAL is not compatible with EV_READ, EV_WRITE or EV_CLOSED", __func__);
            return -1;
        }
        ev->ev_closure = EV_CLOSURE_EVENT_SIGNAL;
    } else {
        if (events & EV_PERSIST) {
            evutil_timerclear(&ev->ev_io_timeout);
            ev->ev_closure = EV_CLOSURE_EVENT_PERSIST;
        } else {
            ev->ev_closure = EV_CLOSURE_EVENT;
        }
    }

    min_heap_elem_init_(ev);
    if (base != NULL) ev->ev_pri = base->nactivequeues / 2;

    event_debug_note_setup_(ev);
    return 0;
}
```

`event_assign()` 实际上就是通过传入的参数对 `event` 进行了设置，并且对一些值进行了初始化：`signal` 事件的激活处理方式为 `EV_CLOSURE_SIGNAL`，永久事件的激活处理方式为 `EV_CLOSURE_PERSIST`，一般的 `IO` 事件激活处理方式则为 `EV_CLOSURE_NONE`。需要注意的是，如果一个 `event` 的感兴趣事件类型设置为了 `EV_SIGNAL`，那么它就不能再同时设置读或写事件为感兴趣事件，反之亦然。

如果是要创建超时事件，`fd` 和 `events` 的值分别设置为 `-1` 和 `0`，`libevent` 用一个宏来表示，超时事件的创建：

``` c++
#define evtimer_new(b, cb, arg) event_new((b), -1, 0, (cb), (arg))
```

此时新创建的 `event` 处于 `non-pending` 状态，`event_assign()` 为其设置超时的时间，这个步骤会推迟到 `event` 被添加到 `reactor` 中的时候。跟着箭头向下，`event_add()` 会将 `non-pending` 的 `event` 添加到 `reactor` 中，同时为其设置超时时间。

- `event_assign()` 中使用了宏来对 `event` 中的参数赋值，所以看起来像赋值了很多 `event` 中没有的成员，部分宏定义如下：

    ``` c++
    #define ev_ncalls       ev_.ev_signal.ev_ncalls
    #define ev_pncalls      ev_.ev_signal.ev_pncalls
    #define ev_pri          ev_evcallback.evcb_pri
    #define ev_flags        ev_evcallback.evcb_flags
    #define ev_closure      ev_evcallback.evcb_closure
    #define ev_callback     ev_evcallback.evcb_cb_union.evcb_callback
    #define ev_arg          ev_evcallback.evcb_arg
    ```

创建好一个 `event` 之后，接着就需要将其添加到 `event_base` 中的相关集合中，以供后续进行监听和处理：

## 添加 event 到 event_base

`libevent` 向用户提供的添加接口是 `event_add()` 函数，实际上这个函数内部主要是调用 `event_add_nolock_()` 函数来完成 `event` 的添加：

``` c++
int event_add(struct event *ev, const struct timeval *tv) {
    int res;

    if (EVUTIL_FAILURE_CHECK(!ev->ev_base)) {
        event_warnx("%s: event has no event_base set.", __func__);
        return -1;
    }

    EVBASE_ACQUIRE_LOCK(ev->ev_base, th_base_lock);
    res = event_add_nolock_(ev, tv, 0);
    EVBASE_RELEASE_LOCK(ev->ev_base, th_base_lock);
    return (res);
}
```

`event_add` 并不做实际的工作，而是获得锁之后，将任务交给 `event_add_nolock_()` 来完成，关于 `libevent` 线程安全的内容暂且不说，后边单独开一篇来讨论。

### event_add_nolock_()

`event_add_nolock_()` 干的活儿非常多，为了深入理解这里也不会一一展开，同样也需要留到后边剖析 (事件管理框架，超时时间管理)，这里仅仅摘出一条主线，省略其他干扰项：

``` c++
int event_add_nolock_(struct event *ev, const struct timeval *tv, int tv_is_absolute) {
    ......
    if ((ev->ev_events & (EV_READ|EV_WRITE|EV_CLOSED|EV_SIGNAL)) &&
        !(ev->ev_flags & (EVLIST_INSERTED|EVLIST_ACTIVE|EVLIST_ACTIVE_LATER))) {
        if (ev->ev_events & (EV_READ|EV_WRITE|EV_CLOSED))
            res = evmap_io_add_(base, ev->ev_fd, ev);
        else if (ev->ev_events & EV_SIGNAL)
            res = evmap_signal_add_(base, (int)ev->ev_fd, ev);
        if (res != -1)
            event_queue_insert_inserted(base, ev);
        if (res == 1) {
            /* evmap says we need to notify the main thread. */
            notify = 1;
            res = 0;
        }
    }
    ......
    event_queue_insert_timeout(base, ev);
    ......
}
```

之前的文章提到过：`libevent` 分别利用两个数组分别管理 `I/O` 事件和信号事件（`Windows` 上使用 `hashmap` 管理 `I/O` 事件），利用 `min_heap` 管理超时事件。`event` 变量正是通过 `event_add_nolock_` 函数被调加到对应的数据结构上，分别是 `evmap_io_add_()`、`evmap_signal_add_()` 和 `event_queue_insert_timeout()` 三个函数。

至此，`event` 的状态就变为了 `pending` 状态，可以通过 `event_pending` 函数获取。事件循环开始工作后，`event` 就处于被监听的状态了。

## 从 event_base 删除 event

`libevent` 同样提供了接口让我们取消监听 `event`，和 `event_add()` 类似，`event_del()` 也是讲任务交给 `event_del_nolock_()` 来完成。取消监听和监听是逆操作，这在代码里边也有体现, 同样地，只摘出了主线：

``` c++
int event_del_nolock_(struct event *ev, int blocking) {
    ......
    if (ev->ev_flags & EVLIST_TIMEOUT) event_queue_remove_timeout(base, ev);

    if (ev->ev_flags & EVLIST_ACTIVE) event_queue_remove_active(base, event_to_event_callback(ev));
    else if (ev->ev_flags & EVLIST_ACTIVE_LATER) event_queue_remove_active_later(base, event_to_event_callback(ev));

    if (ev->ev_flags & EVLIST_INSERTED) {
        event_queue_remove_inserted(base, ev);
        if (ev->ev_events & (EV_READ|EV_WRITE|EV_CLOSED)) res = evmap_io_del_(base, ev->ev_fd, ev);
        else res = evmap_signal_del_(base, (int)ev->ev_fd, ev);
        if (res == 1) {
            notify = 1; /* evmap says we need to notify the main thread. */
            res = 0;
        }
    }
    ......
}
```

因为 `event` 的回调函数可能在其他线程正在运行着，为了线程安全，`blocking` 参数是必要的。

`event_del_nolock_()` 主要做的事情正和 `event_add_nolock_()` 相反，它把定时器从超时事件最小堆上移除，然后将其从信号或者 `I/O` 的 `hashmap` 上移除。额外地，还需要将就绪的回调函数从待处理的回调函数链表上摘除，也就是说如果这个 event 已经触发，顺利调用了 `event_del()` 之后，它的回调函数不会被运行。

`event` 的状态会从 `pending` 回到 `non-pending` 状态。

## 释放 event

和 `event_new()` 对应，`event_free()` 先取消监听 `event`，然后释放其内存：

``` c++
void event_free(struct event *ev) {
    event_del(ev);
    mm_free(ev);
}
```

虽然简单，但是却很重要，尤其不要忘记释放内存，造成内存泄漏。

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [libevent 源码学习（9）：事件 event_HerofH_的博客 - CSDN 博客](https://blog.csdn.net/qq_28114615/article/details/96864601)
* [3] [抽丝剥茧 libevent——初识 event | Baixiangcpp's Blog](http://www.ilovecpp.com/2018/05/01/libevent-event-analyze/)
* [4] [Libevent 深入浅出 · libevent 深入浅出](https://www.gitbook.com/book/aceld/libevent)
