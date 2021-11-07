---
title: 源码阅读 libevent - 结构体：event_once
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: c8f9ab4c
date: 2020-11-04 16:20:40
---
对于只需要触发一次的事件，`libevent` 提供了一种方案：`event_once`，让使用者不需要手动管理 `event`，并且也保证了事件被触发后，内存会被自动被释放。

<!--more-->

## struct event_once 的定义

``` c++
struct event_once {
    LIST_ENTRY(event_once) next_once;
    struct event ev;
    void (*cb)(evutil_socket_t, short, void *);
    void *arg;
};
```

## 创建 event_once

并没有 `event_once_new()` 之类的函数用于创建 `event_once`，要使用它要到一个新的函数 `event_base_once()`, 它将 `event_new()` 和 `event_add()` 两个步骤合成了一个：

``` c++

/* Schedules an event once */
int event_base_once(struct event_base *base, evutil_socket_t fd, short events,
    void (*callback)(evutil_socket_t, short, void *), void *arg, const struct timeval *tv)
{
    ......
    if ((eonce = mm_calloc(1, sizeof(struct event_once))) == NULL) return (-1);
    eonce->cb = callback;
    eonce->arg = arg;

    if ((events & (EV_TIMEOUT|EV_SIGNAL|EV_READ|EV_WRITE|EV_CLOSED)) == EV_TIMEOUT) {
        evtimer_assign(&eonce->ev, base, event_once_cb, eonce);
        if (tv == NULL || ! evutil_timerisset(tv)) activate = 1;
    } else if (events & (EV_READ|EV_WRITE|EV_CLOSED)) {
        events &= EV_READ|EV_WRITE|EV_CLOSED;
        event_assign(&eonce->ev, base, fd, events, event_once_cb, eonce);
    } else {
        mm_free(eonce);
        return (-1);
    }

    if (res == 0) {
        if (activate) event_active_nolock_(&eonce->ev, EV_TIMEOUT, 1);
        else res = event_add_nolock_(&eonce->ev, tv, 0);

        if (res != 0) {
            mm_free(eonce);
            return (res);
        } else {
            LIST_INSERT_HEAD(&base->once_events, eonce, next_once);
        }
    }

    return (0);
}
```

`event_base_once()` 的参数也和 `event_new()` 一样，功能也是一样，不做额外的解释，主要看一下 `event_once()` 里边的多出来的回调函数用来干了什么事情。

我们用 `event_once->cb` 和 `event->cb` 分别表示 `event_once` 和 `event` 里的回调函数。首先函数入口处直接把参数里的 `callback` 赋值给了 `event_once->cb`，它用来负责实际的事件回调。然后调用 `event_assign` 新建一个 `event` 初始化 `event_once->event` 的同时，传入了一个 `event_once_cb()` 回调函数。这个函数才是事件发生时会被回调的函数，也是一次性事件的秘密所在。

## event_once 的回调和销毁

上文中说到，`event_once_cb()` 是一次性事件的真正回调函数，我们来看下该函数的实现：

``` c++
static void event_once_cb(evutil_socket_t fd, short events, void *arg) {
    struct event_once *eonce = arg;

    (*eonce->cb)(fd, events, eonce->arg);
    EVBASE_ACQUIRE_LOCK(eonce->ev.ev_base, th_base_lock);
    LIST_REMOVE(eonce, next_once);
    EVBASE_RELEASE_LOCK(eonce->ev.ev_base, th_base_lock);
    event_debug_unassign(&eonce->ev);
    mm_free(eonce);
}
```

`event_once_cb()` 这个函数被回调时，会马上调用 `event_once->cb`, 然后把他从一次性事件的链表里移除，最后释放掉整个 `event_once` 所分配的内存。

要是这个事件不触发，那么他的回调函数就不会被释放，`event_once` 所占用的内存就得不到释放，我们无法获得它的指针对其 `free`。一次性事件链表就是为了解决这个问题的，他是 `event_base` 里的一个结构。

``` c++
LIST_HEAD(once_event_list, event_once) once_events;
```

所有的 `event_once` 在释放之前都会保留在这个链表里，除了 `event_once_cb()` 触发时会被移除，在 `event_base` 被释放时，也会将所有 `once_events` 里的 `event_once` 逐个释放。

``` c++
static void event_base_free_(struct event_base *base, int run_finalizers) {
    ......
    while (LIST_FIRST(&base->once_events)) {
        struct event_once *eonce = LIST_FIRST(&base->once_events);
        LIST_REMOVE(eonce, next_once);
        mm_free(eonce);
    }
    ......
}
```

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [抽丝剥茧 libevent——初识 event | Baixiangcpp's Blog](http://www.ilovecpp.com/2018/05/01/libevent-event-analyze/)
