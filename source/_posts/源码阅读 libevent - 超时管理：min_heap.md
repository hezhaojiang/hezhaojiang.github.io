---
title: 源码阅读 libevent - 超时管理：min_heap
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: '97886769'
date: 2020-11-06 05:08:22
---
`libevent` 允许创建一个超时 `event`，使用 `evtimer_new` 宏。

``` cpp
#define evtimer_new(b, cb, arg)     event_new((b), -1, 0, (cb), (arg))
```

<!--more-->

从宏的实现来看，它一样是用到了一般的 `event_new`，并且不使用任何的文件描述符。从超时 `event` 宏的实现来看，无论是 `evtimer` 创建的 `event` 还是一般 `event_new` 创建的 `event`，都能使得 `libevent` 进行超时监听。

使得 `libevent` 对一个 `event` 进行超时监听的原因是：在调用 `int event_add(struct event *ev, const struct timeval *tv)` 的时候，第二参数不能为 `NULL`，要设置一个超时值。如果为 `NULL`，那么 `libevent` 将不会为这个 `event` 监听超时。下文统一称设置了超时值的 `event` 为超时 `event`。

## 超时 event 的原理

`libevent` 对超时进行监听的原理不同于之前讲到的对信号的监听，`libevent` 对超时的监听的原理是，多路 `IO` 复用函数都是有一个超时值。如果用户需要 `libevent` 同时监听多个超时 `event`，那么 `libevent` 就把超时值最小的那个作为多路 `IO` 复用函数的超时值。自然，当时间一到，就会从多路 `IO` 复用函数返回。此时对超时 `event` 进行处理即可。

`libevent` 运行用户同时监听多个超时 `event`，那么就必须要对这个超时值进行管理。`libevent` 提供了小根堆 `min_heap` 和通用超时 `common timeout` 两种管理方式。本文首先分析小根堆 `min_heap` 超时管理机制。

## 设置超时值

首先调用 event_add 时要设置一个超时值，这样才能成为一个超时 event。

``` cpp
int event_add(struct event *ev, const struct timeval *tv) {
    ......
    res = event_add_nolock_(ev, tv, 0);
    ......
}

int event_add_nolock_(struct event *ev, const struct timeval *tv, int tv_is_absolute) {
    ...... /* common timeout 相关代码没有展示 */
    /* tv 不为 NULL, 就说明是一个超时 event, 在小根堆中为其留一个位置 */
    if (tv != NULL && !(ev->ev_flags & EVLIST_TIMEOUT)) {
        if (min_heap_reserve_(&base->timeheap, 1 + min_heap_size_(&base->timeheap)) == -1)
            return (-1);  /* ENOMEM == errno */
    }
    ...... /* 将 IO 或者信号 event 插入到对应的队列中 */
    if (res != -1 && tv != NULL) {
        struct timeval now;
        /* 用户把这个 event 设置成 EV_PERSIST。即永久 event 如果没有这样设置的话，那么只会超时一次
        ** 设置了，那么就可以超时多次。那么就要记录用户设置的超时值。 */
        if (ev->ev_closure == EV_CLOSURE_EVENT_PERSIST && !tv_is_absolute)
            ev->ev_io_timeout = *tv;
        /* 因为可以在次线程调用 event_add。而主线程刚好在执行 event_base_dispatch */
        if ((ev->ev_flags & EVLIST_ACTIVE) && (ev->ev_res & EV_TIMEOUT)) {
            if (ev->ev_events & EV_SIGNAL) {
                if (ev->ev_ncalls && ev->ev_pncalls) { /* Abort loop */
                    *ev->ev_pncalls = 0;
                }
            }
            /* 从超时队列中删除这个 event。因为下次会再次加入。多次对同一个超时 event 调用 event_add, 那么只能保留最后的那个。 */
            event_queue_remove_active(base, event_to_event_callback(ev));
        }
        gettime(base, &now); /* 获取现在的时间 */
        /* 虽然用户在 event_add 时只需用一个相对时间，但实际上在 Libevent 内部还是要把这个时间转换成绝对时间。
        ** 从存储的角度来说，存绝对时间只需一个变量。而相对时间则需两个，一个存相对值，另一个存参照物。 */
        if (tv_is_absolute) ev->ev_timeout = *tv; /* 该参数指明时间是否为一个绝对时间 */
        else evutil_timeradd(&now, tv, &ev->ev_timeout); /* 参照时间 + 相对时间  ev_timeout 存的是绝对时间 */
        event_queue_insert_timeout(base, ev); /* 将该超时 event 插入到超时队列中 */
        struct event* top = NULL;
        /* 本次插入的超时值，是所有超时中最小的。那么此时就需要通知主线程。. */
        if (min_heap_elt_is_top_(ev)) notify = 1;
        else if ((top = min_heap_top_(&base->timeheap)) != NULL && evutil_timercmp(&top->ev_timeout, &now, <))
            notify = 1;
    }

    /* 如果代码逻辑中是需要通知的。并且本线程不是主线程。那么就通知主线程 */
    if (res != -1 && notify && EVBASE_NEED_NOTIFY(base)) evthread_notify_base(base);
    return (res);
}
```

对于同一个 `event`，如果是 `IO event` 或者 `sigal event`，那么将无法多次添加。但如果是一个超时 `event`，那么是可以多次添加的。并且对应超时值会使用最后添加时指明的那个，之前的统统不要，即替换掉之前的超时值。

## 调用多路 IO 复用函数等待超时

### event_base_loop

现在来看一下 `event_base_loop` 函数，看其是怎么处理超时 `event` 的。

``` cpp
/* 非超时相关代码没有展示 */
int event_base_loop(struct event_base *base, int flags) {
    const struct eventop *evsel = base->evsel;
    struct timeval tv;
    struct timeval *tv_p;
    int res, done, retval = 0;
    done = 0;
    while (!done) {
        tv_p = &tv;
        if (!N_ACTIVE_CALLBACKS(base) && !(flags & EVLOOP_NONBLOCK)) {
            /* 根据 Timer 事件计算 evsel->dispatch 的最大等待时间（超时值最小） */
            timeout_next(base, &tv_p);
        } else {
            /* if we have active events, we just poll new events without waiting. */
            evutil_timerclear(&tv);
        }
        res = evsel->dispatch(base, tv_p);
        /* 处理超时事件，将超时事件插入到激活链表中 */
        timeout_process(base);
    }

done:
    return (retval);
}
```

`event_base_loop()` 中有关超时管理方面工作不多，最主要的工作有两部分：

1. 设置 `dispatch` 回调的第二个参数 `tv`，这个参数如果为 `0`, 则无论是否有事件发生，都会立即返回。
    1. 如果设置了 `EVLOOP_NONBLOCK` 标志位，则会调用 `evutil_timerclear()` 将 `tv` 设置为 `0`

        ``` cpp
        /* 不会阻塞，它仅仅是查看是否已经有 event ready. 有则运行其 callback. 然后退出 */
        #define EVLOOP_NONBLOCK 0x02

        #define evutil_timerclear(tvp)  (tvp)->tv_sec = (tvp)->tv_usec = 0
        ```

    2. 如果不存在激活的 `event`，也会调用 `evutil_timerclear()` 将 `tv` 设置为 `0`
    3. 如果存在激活的 `event` 且没有设置 `EVLOOP_NONBLOCK` 标志位，则需要调用 `timeout_next()` 获取最近的超时 `event`，并作为等待事件的时间传给 `dispatch` 回调函数

2. 调用 `timeout_process()` 函数将超时了的 `event` 加入激活队列。

### timeout_next

`timeout_next()` 用来计算出本次调用多路 `IO` 复用函数的等待时间：

``` cpp
static int timeout_next(struct event_base *base, struct timeval **tv_p) {
    /* Caller must hold th_base_lock */
    struct timeval now;
    struct event *ev;
    struct timeval *tv = *tv_p;
    int res = 0;
    /* 堆的首元素具有最小的超时值，这个是小根堆的性质。 */
    ev = min_heap_top_(&base->timeheap);
    if (ev == NULL) { *tv_p = NULL; goto out; } /* 堆中没有元素 */
    if (gettime(base, &now) == -1) { res = -1; goto out; } /* /* 获取当前时间 */
    /* 如果超时时间 <= 当前时间，不能等待，需要立即返回
    ** 因为 ev_timeout 这个时间是由 event_add 调用时的绝对时间 + 相对时间。所以 ev_timeout 是绝对时间。
    ** 可能在调用 event_add 之后，过了一段时间才调用 event_base_diapatch, 所以现在可能都过了用户设置的超时时间。 */
    if (evutil_timercmp(&ev->ev_timeout, &now, <=)) { evutil_timerclear(tv); goto out; }
    /* 计算等待的时间 = 当前时间 - 最小的超时时间 */
    evutil_timersub(&ev->ev_timeout, &now, tv);
    event_debug(("timeout_next: event: %p, in %d seconds, %d useconds", ev, (int)tv->tv_sec, (int)tv->tv_usec));
out:
    return (res);
}
```

### timeout_process

`timeout_process()` 函数将超时了的 `event` 加入激活队列：

``` cpp
static void timeout_process(struct event_base *base) {
    struct timeval now;
    struct event *ev;

    if (min_heap_empty_(&base->timeheap)) return;
    gettime(base, &now);

    /* 遍历小根堆的元素。之所以不是只取堆顶那一个元素，是因为当主线程调用多路 IO 复用函数进入等待时，次线程可能添加了多个超时值更小的 event */
    while ((ev = min_heap_top_(&base->timeheap))) {
        /* ev->ev_timeout 存的是绝对时间，超时时间比此刻时间大，说明该 event 还没超时。那么余下的小根堆元素更不用检查了。 */
        if (evutil_timercmp(&ev->ev_timeout, &now, >)) break;
        /* 下面说到的 del 是等同于调用 event_del. 把 event 从这个 event_base 中 (所有的队列都) 删除。event_base 不再监听之。
        ** 这里是 timeout_process 函数。所以对于有超时的 event，才会被 del 掉。
        ** 对于有 EV_PERSIST 选项的 event，在处理激活 event 的时候，会再次添加进 event_base 的。
        ** 这样做的一个好处就是，再次添加的时候，又可以重新计算该 event 的超时时间 (绝对时间)。 */
        event_del_nolock_(ev, EVENT_DEL_NOBLOCK);
        /* 把这个 event 加入到 event_base 的激活队列中。event_base 的激活队列又有该 event 了。
        ** 如果该 event 是 EV_PERSIST 的，可以再次添加进该 event_base */
        event_active_nolock_(ev, EV_TIMEOUT, 1);
    }
}
```

当从多路 `IO` 复用函数返回时，就检查时间小根堆，看有多少个 `event` 已经超时了。如果超时了，那就把这个 `event` 加入到 `event_base` 的激活队列中。并且把这个超时删除掉。

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [libevent-src-analysis/libevent 源码分析. md at master · KelvinYin/libevent-src-analysis](https://github.com/KelvinYin/libevent-src-analysis/blob/master/libevent%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
