---
title: 源码阅读 libevent - 工作流程分析
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: dd5b304b
date: 2020-11-09 04:36:32
---
之前的小节讲了很多 `libevent` 的基础构件，现在以一个实际例子来初步探究 `libevent` 的基本工作流程。

<!--more-->

## libevent 工作流程分析

``` cpp
#include<stdio.h>
#include<event.h>
#include<thread.h>

void cmd_cb(int fd, short events, void *arg) {
    char buf[1024];
    printf("in the cmd_cb\n");
    read(fd, buf, sizeof(buf));
}

int main() {
    evthread_use_pthreads();
    // 使用默认的 event_base 配置
    struct event_base *base = event_base_new();
    struct event *cmd_ev = event_new(base, STDIN_FILENO,
                                     EV_READ | EV_PERSIST, cmd_cb, NULL);
    event_add(cmd_ev, NULL); // 没有超时
    event_base_dispatch(base);
    return 0;
}
```

这个例子非常简单，却已经包含了 `libevent` 的基础工作流程：

1. `event_base_new()`
2. `event_new()`
3. `event_add()`
4. `event_base_dispatch()`

### evthread_use_pthreads

`evthread_use_pthreads()` 开启多线程支持。

> 参见：{% post_link "源码阅读 libevent - 多线程：锁和条件变量" %}

### event_base_new

`event_base_new()` 构造一个 `event_base` 对象。

> 参见：{% post_link "源码阅读 libevent - 创建 event_base" %}

### event_new

`event_new()` 创建一个 `event` 事件。

### event_add

`event_add()` 将 `event` 事件加入到 `event_base` 对象中。

> 参见：{% post_link "源码阅读 libevent - 结构体：event" %}

### event_base_dispatch

`event_base_dispatch()` 开始监听 `event` 事件发生。

> 参见：{% post_link "源码阅读 libevent - 事件主循环" %}

## 源码学习流程

### 该博客文章建议阅读顺序

- {% post_link "源码阅读 libevent - 初识 Hello World" %}
- {% post_link "源码阅读 libevent - 日志模块" %}
- {% post_link "源码阅读 libevent - 内存管理" %}
- {% post_link "源码阅读 libevent - 多线程：锁和条件变量" %}
- {% post_link "源码阅读 libevent - 多线程：调试锁" %}
- {% post_link "源码阅读 libevent - 数据结构：双向链表" %}
- {% post_link "源码阅读 libevent - 数据结构：尾队列" %}
- {% post_link "源码阅读 libevent - 数据结构：哈希表" %}
- {% post_link "源码阅读 libevent - 结构体：event_io_map" %}
- {% post_link "源码阅读 libevent - 结构体：event_signal_map" %}
- {% post_link "源码阅读 libevent - 创建 event_base" %}
- {% post_link "源码阅读 libevent - 结构体：event" %}
- {% post_link "源码阅读 libevent - 结构体：event_once" %}
- {% post_link "源码阅读 libevent - 事件主循环" %}
- {% post_link "源码阅读 libevent - 数据结构：小根堆" %}
- {% post_link "源码阅读 libevent - 超时管理：min_heap" %}
- {% post_link "源码阅读 libevent - 信号事件处理" %}
- {% post_link "源码阅读 libevent - 工作流程分析" %}
- {% post_link "源码阅读 libevent - 优先级管理" %}
- {% post_link "源码阅读 libevent - 时间管理" %}

### 待完成文章

因为有新系列文章计划，该系列文章仍会持续更新，但不会很频繁了。

- 源码阅读 libevent - 超时管理：common timeout
- 源码阅读 libevent - 通知机制
- 未完待续

### 其他资料整理

本文是我在阅读源码过程中，对自己学习情况的一个记录，学习过程中参考了很多资料，也对所有资料的作者表示感谢：

* [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [libevent-src-analysis/libevent 源码分析. md at master · KelvinYin/libevent-src-analysis](https://github.com/KelvinYin/libevent-src-analysis/blob/master/libevent%E6%BA%9090%E7%A0%81%E5%88%86%E6%9E%90.md)
