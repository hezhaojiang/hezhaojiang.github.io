---
title: 源码阅读 libevent - 事件主循环
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: f65e9050
date: 2020-11-09 05:03:49
---
在 `libevent` 中，事件主循环的作用就是执行一个循环，在循环中监听事件以及超时的事件并且将这些激活的事件进行处理。`libevent` 提供了对用户开放了两种执行事件主循环的函数：

``` c++
int event_base_dispatch(struct event_base *);
int event_base_loop(struct event_base *, int);
```

<!--more-->

## 事件主循环

> {% post_link "源码阅读 libevent - 结构体：event" %} 中提到了 `event` 结构体的生命周期:

> ![event 的生命周期](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201128204050.png)

事件主循环做的工作就是该流程图的下半部分：

pending 状态：等待事件发生
active 状态：事件已经发生，等待调用事件回调

总体流程分为三部分

| 流程 | 调用相关函数 | event 状态 |
| - | - | - |
| 事件发生 | event_active() | pending --> avtive |
| 单次事件处理 | callback done | avtive --> non-pending |
| 持久事件处理 | callback done | avtive --> pending |

事件主循环函数 `event_base_dispatch()` 其实就是调用 `event_base_loop()`：

``` c++
int event_base_dispatch(struct event_base *event_base) {
    return (event_base_loop(event_base, 0));
}
```

`event_base_loop()` 函数实现如下：

``` c++
int event_base_loop(struct event_base *base, int flags) {
    const struct eventop *evsel = base->evsel;
    struct timeval tv;
    struct timeval *tv_p;
    int res, done, retval = 0;
    /* 加锁. 在调用 evsel->dispatch 时会进行解锁, 在调用用户回调函数时也会解锁 */
    EVBASE_ACQUIRE_LOCK(base, th_base_lock);
    if (base->running_loop) {
        event_warnx("%s: reentrant invocation.  Only one event_base_loop can run on each event_base at once.", __func__);
        EVBASE_RELEASE_LOCK(base, th_base_lock);
        return -1;
    }
    base->running_loop = 1; /* 只允许一个事件主循环 */

    clear_time_cache(base); /* 清空缓存的超时 */
    if (base->sig.ev_signal_added && base->sig.ev_n_signals_added) evsig_set_base_(base);
    done = 0; /* 用来确定是否退出循环 */
#ifndef EVENT__DISABLE_THREAD_SUPPORT
    base->th_owner_id = EVTHREAD_GET_ID(); /* 保存当前线程 ID */
#endif
    base->event_gotterm = base->event_break = 0;

    while (!done) {
        base->event_continue = 0;
        base->n_deferreds_queued = 0;

        /* 中止循环的两个条件 */
        if (base->event_gotterm) break;
        if (base->event_break) break;
        tv_p = &tv;
        /* 如果当前 event_base 里没有已经激活的事件，就将时间最小堆，堆顶的超时值取出来，作为下一轮 dispatch 的超时值
        ** 否则就将超时时间置为 0，evsel->dispatch 会立马超时返回，激活的事件得以处理 */
        if (!N_ACTIVE_CALLBACKS(base) && !(flags & EVLOOP_NONBLOCK)) timeout_next(base, &tv_p);
        else evutil_timerclear(&tv);
        /* If we have no events, we just exit */
        if (0 == (flags & EVLOOP_NO_EXIT_ON_EMPTY) &&
            !event_haveevents(base) && !N_ACTIVE_CALLBACKS(base)) {
            event_debug(("%s: no events registered.", __func__));
            retval = 1;
            goto done;
        }
        event_queue_make_later_events_active(base);
        clear_time_cache(base);
        res = evsel->dispatch(base, tv_p);
        if (res == -1) {
            event_debug(("%s: dispatch returned unsuccessfully.", __func__));
            retval = -1;
            goto done;
        }

        update_time_cache(base);
        /* 将 base 的 min_heap 中所有超时的事件以超时激活类型添加到激活队列中 */
        timeout_process(base);
        if (N_ACTIVE_CALLBACKS(base)) { /* 如果激活队列中有事件 */
            /* 执行激活队列中的 event 相应的回调函数，返回的 n 是成功执行的非内部事件数目 */
            int n = event_process_active(base);
            /* 如果设置了 EVLOOP_ONCE，并且所有激活的事件都处理完了，那么就退出 event_loop */
            if ((flags & EVLOOP_ONCE) && N_ACTIVE_CALLBACKS(base) == 0 && n != 0)
                done = 1;
        } else if (flags & EVLOOP_NONBLOCK) /* 如果设置了 EVLOOP_NONBLOCK 那么也会退出 event_loop 循环 */
            done = 1;
    }
    event_debug(("%s: asked to terminate loop.", __func__));

done:
    clear_time_cache(base);
    base->running_loop = 0;
    EVBASE_RELEASE_LOCK(base, th_base_lock);
    return (retval);
}
```

直接阅读 `event_base_loop()` 较为困难，我们根据上述流程进行分析

1. 事件发生
    - 如何监听事件发生
    - 事件发生后如何激活事件
2. 事件处理
    - 如何处理已激活事件
    - 如何持久化事件

## 事件发生

### 如何监听事件发生

#### 监听方式

libevent 提供了多种监听事件的方案，如单次监听、循环监听等，监听方案由 `event_base_loop()` 参数决定：

``` c++
    /* 阻塞, 直到一个 event 变成 active. 在 active 状态的 event 的 Callback 函数执行后，就退出。 */
    #define EVLOOP_ONCE 0x01
    /* 不会阻塞，它仅仅是查看是否已经有 event ready. 有则运行其 callback. 然后退出 */
    #define EVLOOP_NONBLOCK 0x02
    /* 哪怕 event_base 中没有 active 或者 pending 的 event 了。也不退出。直到调用 event_base_loopbreak() or event_base_loopexit(). */
    #define EVLOOP_NO_EXIT_ON_EMPTY 0x04

    /* 当 flags 什么都不设置时，则 loop 一直运行，检查到 event 变为 active 时，运行其 callback 函数。
    ** 当没有 active 或 pending 的 event 后，退出 loop。
    ** 有人调用了 event_base_loopbreak() or event_base_loopexit()，也退出 loop. */
```

> 有关 event_base_loopbreak() 和 event_base_loopexit() 参见：{% post_link "源码阅读 libevent - 事件主循环的停止" %}

#### 监听超时时间的计算

#### 监听 IO 事件和 Signal 事件

`libevent` 调用 `evsel->dispatch` 回调函数监听 `IO` 事件和 `Signal` 事件的发生：

以 `select.c` 为例：

#### select

``` c++
static int select_dispatch(struct event_base *base, struct timeval *tv) {
    int res=0, i, j, nfds;
    struct selectop *selectop = base->evbase;

    if (selectop->resize_out_sets) {
        fd_set *readset_out = NULL, *writeset_out = NULL;
        size_t sz = selectop->event_fdsz;
        if (!(readset_out = mm_realloc(selectop->event_readset_out, sz))) return (-1);
        selectop->event_readset_out = readset_out;
        if (!(writeset_out = mm_realloc(selectop->event_writeset_out, sz))) {
            /* We don't free readset_out here, since it was already successfully reallocated.
            ** The next time we call select_dispatch, the realloc will be a no-op. */
            return (-1);
        }
        selectop->event_writeset_out = writeset_out;
        selectop->resize_out_sets = 0;
    }
    memcpy(selectop->event_readset_out, selectop->event_readset_in, selectop->event_fdsz);
    memcpy(selectop->event_writeset_out, selectop->event_writeset_in, selectop->event_fdsz);
    nfds = selectop->event_fds + 1;

    EVBASE_RELEASE_LOCK(base, th_base_lock);
    res = select(nfds, selectop->event_readset_out, selectop->event_writeset_out, NULL, tv);
    EVBASE_ACQUIRE_LOCK(base, th_base_lock);

    if (res == -1) {
        if (errno != EINTR) return (-1);
        return (0);
    }
    event_debug(("%s: select reports %d", __func__, res));

    i = evutil_weakrand_range_(&base->weakrand_seed, nfds);
    for (j = 0; j < nfds; ++j) {
        if (++i>= nfds) i = 0;
        res = 0;
        if (FD_ISSET(i, selectop->event_readset_out)) res |= EV_READ;
        if (FD_ISSET(i, selectop->event_writeset_out)) res |= EV_WRITE;
        if (res == 0) continue;
        evmap_io_active_(base, i, res);
    }
    return (0);
}
```

在事件发生后调用 `evmap_io_active_()` 将发生的事件加入到激活事件队列中，参见 [事件发生后如何激活事件](# 事件发生后如何激活事件)。

### 事件发生后如何激活事件

三种事件的激活方式如下图所示：

![事件激活](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201111125131.png)

#### 激活 IO 事件

##### evmap_io_active_

从 `select_dispatch` 函数中可以看出，对于每一个事件发生的 `fd`，均会调用一次 `evmap_io_active_` 函数。

``` c++
void evmap_io_active_(struct event_base *base, evutil_socket_t fd, short events) {
    struct event_io_map *io = &base->io;
    struct evmap_io *ctx;
    struct event *ev;

    if (fd < 0 || fd>= io->nentries) return;
    /* 在 IO 事件的哈希表中获取该 fd 已添加的事件 */
    GET_IO_SLOT(ctx, io, fd, evmap_io);

    if (NULL == ctx) return;
    /* 遍历哈希表中所有该 fd 已添加的事件 */
    LIST_FOREACH(ev, &ctx->events, ev_io_next) {
        /* 比对 dispatch 的事件和已添加的事件 */
        if (ev->ev_events & events) s(ev, ev->ev_events & events, 1);
    }
}
```

##### event_active_nolock_

`evmap_io_active_` 对 `dispatch` 的所有事件进行过滤后，对于所有已添加事件，就需要调用 `event_active_nolock_` 进行激活。

``` c++
void event_active_nolock_(struct event *ev, int res, short ncalls) {
    struct event_base *base;
    base = ev->ev_base;
    EVENT_BASE_ASSERT_LOCKED(base);
    /* #define ev_flags ev_evcallback.evcb_flags */
    if (ev->ev_flags & EVLIST_FINALIZING) return;
    /* event 是否在激活链表和下一次激活链表中 */
    switch ((ev->ev_flags & (EVLIST_ACTIVE|EVLIST_ACTIVE_LATER))) {
    default:
    case EVLIST_ACTIVE|EVLIST_ACTIVE_LATER:
        EVUTIL_ASSERT(0);
        break;
    case EVLIST_ACTIVE: /* We get different kinds of events, add them together */
        ev->ev_res |= res;
        return;
    case EVLIST_ACTIVE_LATER:
        ev->ev_res |= res;
        break;
    case 0:
        ev->ev_res = res;
        break;
    }
    if (ev->ev_pri < base->event_running_priority) base->event_continue = 1;
    /* 对于 signal 事件，需要对 IO 事件触发次数进行计数 */
    if (ev->ev_events & EV_SIGNAL) {
        ev->ev_ncalls = ncalls;
        ev->ev_pncalls = NULL;
    }
    event_callback_activate_nolock_(base, event_to_event_callback(ev));
}
```

##### event_callback_activate_nolock_

``` c++
int event_callback_activate_nolock_(struct event_base *base, struct event_callback *evcb) {
    int r = 1;
    if (evcb->evcb_flags & EVLIST_FINALIZING) return 0;
    /* event 是否在激活链表和下一次激活链表中 */
    switch (evcb->evcb_flags & (EVLIST_ACTIVE|EVLIST_ACTIVE_LATER)) {
    default:
        EVUTIL_ASSERT(0);
        EVUTIL_FALLTHROUGH;
    case EVLIST_ACTIVE_LATER:
        event_queue_remove_active_later(base, evcb); /* 删除下一次激活列表中的该 event */
        r = 0;
        break;
    case EVLIST_ACTIVE:
        return 0;
    case 0:
        break;
    }
    /* 没有在激活列表和在下一次激活列表中的 event 会走到这里 */
    event_queue_insert_active(base, evcb); /* 加入激活队列 */
    if (EVBASE_NEED_NOTIFY(base))evthread_notify_base(base);
    return r;
}
```

##### event_queue_insert_active

``` c++
static void
event_queue_insert_active(struct event_base *base, struct event_callback *evcb)
{
    EVENT_BASE_ASSERT_LOCKED(base);
    /* Double insertion is possible for active events */
    if (evcb->evcb_flags & EVLIST_ACTIVE) return;
    INCR_EVENT_COUNT(base, evcb->evcb_flags); /* 非内部事件计数自增 increase */
    evcb->evcb_flags |= EVLIST_ACTIVE; /* event 事件增加激活标志位 */

    base->event_count_active++; /* 激活事件计数自增 */
    MAX_EVENT_COUNT(base->event_count_active_max, base->event_count_active);
    /* #define ev_pri ev_evcallback.evcb_pri 事件优先级 */
    EVUTIL_ASSERT(evcb->evcb_pri < base->nactivequeues);
    /* 插入到对应优先级的激活队列尾部 */
    TAILQ_INSERT_TAIL(&base->activequeues[evcb->evcb_pri], evcb, evcb_active_next);
}
```

#### 激活 Signal 事件

对于 `Signal` 事件，`libevent` 将其统一转换为了 `IO` 事件，在 `IO` 事件中，有一个特殊的事件专门用来接收信号事件，该特殊事件的回调函数会调用 `evmap_signal_active_` 将发生的 `Signal` 事件添加到激活事件列表中。

> 参见：{% post_link "源码阅读 libevent - 信号事件处理" %}

`evmap_signal_active_` 代码如下，其最终也是调用了 `event_active_nolock_` 进行事件的激活。

``` c++
void evmap_signal_active_(struct event_base *base, evutil_socket_t sig, int ncalls) {
    struct event_signal_map *map = &base->sigmap;
    struct evmap_signal *ctx;
    struct event *ev;

    if (sig < 0 || sig>= map->nentries) return;
    /* 在 Signal 事件的数组中获取该 Signal 已添加的事件 */
    GET_SIGNAL_SLOT(ctx, map, sig, evmap_signal);
    if (!ctx) return;
    /* 将该 Signal 已添加的事件全部加入到激活事件列表中 */
    LIST_FOREACH(ev, &ctx->events, ev_signal_next)
        event_active_nolock_(ev, EV_SIGNAL, ncalls);
}
```

#### 激活超时事件

遍历检查小根堆中每个事件是否超时，如果超时，则将其加入到激活队列中，激活事件调用的函数也为 `event_active_nolock_`。

``` c++
static void timeout_process(struct event_base *base) {
    /* Caller must hold lock. */
    struct timeval now;
    struct event *ev;
    if (min_heap_empty_(&base->timeheap)) return;
    gettime(base, &now);
    while ((ev = min_heap_top_(&base->timeheap))) {
        /* 根据小根堆的特性，如果顶部的事件没有超时，其他事件就不用再遍历了 */
        if (evutil_timercmp(&ev->ev_timeout, &now, >)) break;
        /* delete this event from the I/O queues */
        event_del_nolock_(ev, EVENT_DEL_NOBLOCK);
        event_debug(("timeout_process: event: %p, call %p", ev, ev->ev_callback));
        event_active_nolock_(ev, EV_TIMEOUT, 1);
    }
}
```

#### 总结

- `IO` 事件的激活是在 `libevent` 调用多路 `IO` 复用函数后，然后将发生的事件添加入激活队列
- `Signal` 事件的激活在 `libevent` 处理 `IO` 事件的回调中
- 超时事件的激活在 `libevent` 调用完多路 `IO` 复用函数后，检查小根堆里的超时情况时

## 单次事件处理

### 如何处理已激活事件

#### event_process_active

``` c++
static int event_process_active(struct event_base *base) {
    /* Caller must hold th_base_lock */
    struct evcallback_list *activeq = NULL;
    int i, c = 0;
    const struct timeval *endtime;
    struct timeval tv;
    const int maxcb = base->max_dispatch_callbacks;
    const int limit_after_prio = base->limit_callbacks_after_prio;

    if (base->max_dispatch_time.tv_sec >= 0) {
        update_time_cache(base);
        gettime(base, &tv);
        evutil_timeradd(&base->max_dispatch_time, &tv, &tv);
        endtime = &tv;
    } else endtime = NULL;

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
done:
    base->event_running_priority = -1;
    return c;
}
```

#### event_process_active_single_queue

``` c++
static int event_process_active_single_queue(struct event_base *base,
    struct evcallback_list *activeq, int max_to_process, const struct timeval *endtime) {
    struct event_callback *evcb;
    int count = 0;
    EVUTIL_ASSERT(activeq != NULL);
    /* activeq 为某一优先级的激活队列，该处遍历该优先级的激活队列中的所有事件 */
    /* 从遍历结束的结束条件和遍历下一个事件的方式就可知，每次遍历会在激活队列中删除当前事件 */
    for (evcb = TAILQ_FIRST(activeq); evcb; evcb = TAILQ_FIRST(activeq)) {
        struct event *ev = NULL;
        /* 在激活队列中删除当前事件 */
        if (evcb->evcb_flags & EVLIST_INIT) {
            /* 激活队列中仅插入了 event 结构体中的 event_callback 结构体，需要获取 event 所在地址 */
            ev = event_callback_to_event(evcb);
            if (ev->ev_events & EV_PERSIST || ev->ev_flags & EVLIST_FINALIZING)
                event_queue_remove_active(base, evcb); /* 永久事件从激活队列中删除 */
            else
                event_del_nolock_(ev, EVENT_DEL_NOBLOCK); /* 非永久事件从所有队列中删除 */
        } else event_queue_remove_active(base, evcb);

        if (!(evcb->evcb_flags & EVLIST_INTERNAL)) ++count; /* 非内部事件回调次数计数 */
        base->current_event = evcb;

        switch (evcb->evcb_closure) { /* 在调用回调函数是否进行其他行为 */
        case EV_CLOSURE_EVENT_SIGNAL:
            EVUTIL_ASSERT(ev != NULL);
            event_signal_closure(base, ev);
            break;
        case EV_CLOSURE_EVENT_PERSIST: /* 对于永久事件，在调用回调函数之前会重新调用 event_add 来添加该事件到对应队列中 */
            EVUTIL_ASSERT(ev != NULL);
            event_persist_closure(base, ev);
            break;
        case EV_CLOSURE_EVENT: /* 对于一般事件，直接调用回调函数 */
            void (*evcb_callback)(evutil_socket_t, short, void *);
            short res;
            EVUTIL_ASSERT(ev != NULL);
            evcb_callback = *ev->ev_callback;
            res = ev->ev_res;
            EVBASE_RELEASE_LOCK(base, th_base_lock);
            evcb_callback(ev->ev_fd, res, ev->ev_arg);
            break;
        case EV_CLOSURE_CB_SELF:
            void (*evcb_selfcb)(struct event_callback *, void *) = evcb->evcb_cb_union.evcb_selfcb;
            EVBASE_RELEASE_LOCK(base, th_base_lock);
            evcb_selfcb(evcb, evcb->evcb_arg);
            break;
        case EV_CLOSURE_EVENT_FINALIZE:
        case EV_CLOSURE_EVENT_FINALIZE_FREE:
            void (*evcb_evfinalize)(struct event *, void *);
            int evcb_closure = evcb->evcb_closure;
            EVUTIL_ASSERT(ev != NULL);
            base->current_event = NULL;
            evcb_evfinalize = ev->ev_evcallback.evcb_cb_union.evcb_evfinalize;
            EVUTIL_ASSERT((evcb->evcb_flags & EVLIST_FINALIZING));
            EVBASE_RELEASE_LOCK(base, th_base_lock);
            evcb_evfinalize(ev, ev->ev_arg);
            event_debug_note_teardown_(ev);
            if (evcb_closure == EV_CLOSURE_EVENT_FINALIZE_FREE) mm_free(ev);
            break;
        case EV_CLOSURE_CB_FINALIZE:
            void (*evcb_cbfinalize)(struct event_callback *, void *) = evcb->evcb_cb_union.evcb_cbfinalize;
            base->current_event = NULL;
            EVUTIL_ASSERT((evcb->evcb_flags & EVLIST_FINALIZING));
            EVBASE_RELEASE_LOCK(base, th_base_lock);
            evcb_cbfinalize(evcb, evcb->evcb_arg);
            break;
        default:
            EVUTIL_ASSERT(0);
        }

        EVBASE_ACQUIRE_LOCK(base, th_base_lock);
        base->current_event = NULL;

        if (base->event_break) return -1;
        if (count>= max_to_process) return count;
        if (count && endtime) {
            struct timeval now;
            update_time_cache(base);
            gettime(base, &now);
            if (evutil_timercmp(&now, endtime,>=)) return count;
        }
        if (base->event_continue) break;
    }
    return count;
}
```

#### 如何持久化事件

`libevent` 持久化事件是在调用事件的回调函数之前，调用 `event_add_nolock_` 重新将事件添加到事件列表中：

``` c++
/* Closure function invoked when we're activating a persistent event. */
static inline void event_persist_closure(struct event_base *base, struct event *ev) {
    void (*evcb_callback)(evutil_socket_t, short, void *);
    // Other fields of *ev that must be stored before executing
    evutil_socket_t evcb_fd;
    short evcb_res;
    void *evcb_arg;

    /* reschedule the persistent event if we have a timeout. */
    if (ev->ev_io_timeout.tv_sec || ev->ev_io_timeout.tv_usec) {
        /* If there was a timeout, we want it to run at an interval of ev_io_timeout after the last time
        ** it was _scheduled_ for, not ev_io_timeout after _now_.  If it fired for another reason,
        ** though,the timeout ought to start ticking _now_. */
        struct timeval run_at, relative_to, delay, now;
        ev_uint32_t usec_mask = 0;
        EVUTIL_ASSERT(is_same_common_timeout(&ev->ev_timeout, &ev->ev_io_timeout));
        gettime(base, &now);
        if (is_common_timeout(&ev->ev_timeout, base)) {
            delay = ev->ev_io_timeout;
            usec_mask = delay.tv_usec & ~MICROSECONDS_MASK;
            delay.tv_usec &= MICROSECONDS_MASK;
            if (ev->ev_res & EV_TIMEOUT) {
                relative_to = ev->ev_timeout;
                relative_to.tv_usec &= MICROSECONDS_MASK;
            } else relative_to = now;
        } else {
            delay = ev->ev_io_timeout;
            if (ev->ev_res & EV_TIMEOUT) relative_to = ev->ev_timeout;
            else relative_to = now;
        }
        evutil_timeradd(&relative_to, &delay, &run_at);
        if (evutil_timercmp(&run_at, &now, <)) {
            /* Looks like we missed at least one invocation due to a clock jump, not running the event
            ** loop for a while, really slow callbacks, or something. Reschedule relative to now.*/
            evutil_timeradd(&now, &delay, &run_at);
        }
        run_at.tv_usec |= usec_mask;
        event_add_nolock_(ev, &run_at, 1);
    }

    // Save our callback before we release the lock
    evcb_callback = ev->ev_callback;
    evcb_fd = ev->ev_fd;
    evcb_res = ev->ev_res;
    evcb_arg = ev->ev_arg;

    // Release the lock
    EVBASE_RELEASE_LOCK(base, th_base_lock);
    // Execute the callback
    (evcb_callback)(evcb_fd, evcb_res, evcb_arg);
}
```

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [libevent 源码学习（13）：事件主循环 event_base_loop_HerofH_的博客 - CSDN 博客](https://blog.csdn.net/qq_28114615/article/details/96826553)
