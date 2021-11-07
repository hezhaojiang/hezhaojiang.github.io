---
title: 源码阅读 libevent - 优先级管理
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 27de5435
date: 2020-12-30 13:15:02
---
`event_base` 允许用户对它里面的 `event` 设置优先级，这样可以使得有些更重要的 `event` 能够得到优先处理。

<!--more-->

## 实现优先级

`libevent` 实现优先级功能的方法是：用一个激活队列数组来存放激活 `event`。即数组的元素是一个激活队列，所以有多个激活队列。并且规定不同的队列有不同的优先级。

可以通过 `event_base_priority_init` 函数设置 `event_base` 的优先级个数，该函数实现如下：

``` c++
int event_base_priority_init(struct event_base *base, int npriorities) {
    int i, r = -1;
    EVBASE_ACQUIRE_LOCK(base, th_base_lock);
    if (N_ACTIVE_CALLBACKS(base) || npriorities < 1 || npriorities >= EVENT_MAX_PRIORITIES) goto err;
    if (npriorities == base->nactivequeues) goto ok;
    if (base->nactivequeues) {
        mm_free(base->activequeues);
        base->nactivequeues = 0;
    }
    /* Allocate our priority queues */
    base->activequeues = (struct evcallback_list *)mm_calloc(npriorities, sizeof(struct evcallback_list));
    if (base->activequeues == NULL) {
        event_warn("%s: calloc", __func__);
        goto err;
    }
    base->nactivequeues = npriorities;
    for (i = 0; i < base->nactivequeues; ++i) TAILQ_INIT(&base->activequeues[i]);

ok:
    r = 0;
err:
    EVBASE_RELEASE_LOCK(base, th_base_lock);
    return (r);
}
```

从前面一个判断可知，因为 `event_base_dispatch` 函数会改动激活事件的个数，即会使得 `N_ACTIVE_CALLBACKS(base)` 为真。所以 `event_base_priority_init` 函数要在 `event_base_dispatch` 函数之前调用。此外要设置的优先级个数，要小于 `EVENT_MAX_PRIORITIES`。这个宏是在 `event.h` 文件中定义，在 `2.1.11` 版本中，该宏被定义成 `256`。在调用 `event_base_new` 得到的 `event_base` 只有一个优先级，也就是所有 `event` 都是同级的。

上面的代码调用 `mm_alloc` 分配了一个优先级数组。不同优先级的 `event` 会被放到数组的不同位置上 (下面可以看到这一点)。这样就可以区分不同 `event` 的优先级了。以后处理 `event` 时，就可以从高优先级到低优先级处理 `event`。

## 设置优先级

上面是设置 `event_base` 的优先级个数。现在来看一下怎么设置 `event` 的优先级。可以通过 `event_priority_set` 函数设置，该函数如下：

``` c++
/* Set's the priority of an event - if an event is already scheduled changing the priority is going to fail. */
int event_priority_set(struct event *ev, int pri) {
    event_debug_assert_is_setup_(ev);
    if (ev->ev_flags & EVLIST_ACTIVE) return (-1);
    if (pri < 0 || pri>= ev->ev_base->nactivequeues) return (-1);
    ev->ev_pri = pri;
    return (0);
}
```

在上面代码的第一个判断中，可以知道当 `event` 的状态是 `EVLIST_ACTIVE` 时，就不能对这个 `event` 进行优先级设置了。因此，如果要对 `event` 进行优先级设置，那么得在调用 `event_base_dispatch` 函数之前。因为一旦调用了 `event_base_dispatch`，那么 `event` 就随时可能变成 `EVLIST_ACTIVE` 状态。

## 按优先级激活队列

现在看一下一个 `event` 是怎么插入到 `event_base` 的优先级数组中。

``` c++
static void event_queue_insert_active(struct event_base *base, struct event_callback *evcb)
{
    ... ...
    /* #define ev_pri ev_evcallback.evcb_pri 事件优先级 */
    EVUTIL_ASSERT(evcb->evcb_pri < base->nactivequeues);
    /* 插入到对应优先级的激活队列尾部 */
    TAILQ_INSERT_TAIL(&base->activequeues[evcb->evcb_pri], evcb, evcb_active_next);
}
```

`event` 插入到 `event_base` 的优先级数组中后，会被按照优先级顺序被调用：

``` c++
static int event_process_active(struct event_base *base) {
    ... ...
    /* 按优先级遍历激活队列中的事件 */
    for (i = 0; i < base->nactivequeues; ++i) {
        if (TAILQ_FIRST(&base->activequeues[i]) != NULL) { /* 同一个优先级下可以有多个事件 */
            base->event_running_priority = i; /* 设置当前的优先级 */
            activeq = &base->activequeues[i]; /* 获取优先级 i 下的所有 event 组成的链表 */
            /* 遍历 activeq 链表，调用其中每个 event 的回调函数 */
            if (i < limit_after_prio) c = event_process_active_single_queue(base, activeq, INT_MAX, NULL);
            else c = event_process_active_single_queue(base, activeq, maxcb, endtime);
            if (c < 0) goto done; /* c 是执行的非内部事件数目 */
            else if (c> 0) /* Processed a real event; do not consider lower-priority events */
                break; /* If we get here, all of the events we processed were internal. Continue. */
        }
    }
    ... ...
}
```

## 默认优先级

默认优先级是在新建 `event` 结构体时设置的。不错，看下面的 `event_assign` 函数。

``` c++

int event_assign(struct event *ev, struct event_base *base, evutil_socket_t fd,
                 short events, void (*callback)(evutil_socket_t, short, void *), void *arg) {
    ... ...
    if (base != NULL) {
        /* by default, we put new events into the middle priority */
        ev->ev_pri = base->nactivequeues / 2;
    }
    ... ...
}
```

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [Libevent源码分析 event优先级设置 luotuo44的专栏 - CSDN博客](https://blog.csdn.net/luotuo44/article/details/38512719)
