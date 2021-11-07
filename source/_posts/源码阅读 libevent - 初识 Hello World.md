---
title: 源码阅读 libevent - 初识 Hello World
categories:
    - 源码阅读
tags:
  - libevent
abbrlink: a867e274
date: 2020-05-19 15:22:36
---

## 初识 libevent

### libevent 概述

`Libevent` 有几个显著的亮点：

- 事件驱动（`event-driven`），高性能;
- 轻量级，专注于网络，不如 ACE 那么臃肿庞大；
- 源代码相当精炼、易读；
- 跨平台，支持 Windows、Linux、*BSD 和 Mac Os；
- 支持多种 `I/O` 多路复用技术， `epoll`、`poll`、`dev/poll`、`select` 和 `kqueue` 等；
- 支持 `I/O`，定时器和信号等事件；
- 注册事件优先级；

`Libevent` 已经被广泛的应用，作为底层的网络库；比如 `memcached`、`Vomit`、`Nylon`、`Netchat` 等等。

<!-- more -->

> [libevent 源码深度剖析](https://leanote.com/api/file/getAttach?fileId=577e1ad9ab644133ed002c8a) page5

## 从 Hello World 开始

`hello-world` 是 `libevent` 自带的一个例子，这个例子的作用是启动后监听一个端口，对于所有通过这个端口连接上服务器的程序发送一段字符：hello-world，然后关闭连接。

> 源码链接: <https://github.com/libevent/libevent/blob/master/sample/hello-world.c>

下面就通过分析这个例子来看一下 libevent 对于 IO 事件是如何处理的：

### 模块代码分析

1、调用 `event_base_new` 获得 `event_base` 对象

``` cpp
base = event_base_new();
if (!base) {
    fprintf(stderr, "Could not initialize libevent!\n");
    return 1;
}
```

2、调用 `evconnlistener_new_bind`，返回一个 `struct evconnlistener` 对象指针

``` cpp
listener = evconnlistener_new_bind(
    base,                                           // 所属的 event_base 对象
    listener_cb,                                    // 监听回调函数
    (void *)base,                                   // 回调函数的参数
    LEV_OPT_REUSEABLE|LEV_OPT_CLOSE_ON_FREE,        // 标志：地址可复用，调用 exec 的时候关闭套接字
    -1,                                             // 是否在后台打印日志
    (struct sockaddr*)&sin,                         // 地址
    sizeof(sin)                                     // 地址长度
    );

if (!listener) {
    fprintf(stderr, "Could not create a listener!\n");
    return 1;
}
```

3、调用 `evsignal_new` 函数获取一个信号事件对象, 并将事件处理器添加到 `event_base` 的事件处理器注册队列中

``` cpp
signal_event = evsignal_new(
    base,                       // 所属的 event_base 对象
    SIGINT,                     // 信号处理器处理的信号 这里处理的信号是 SIGINT
    signal_cb,                  // 处理信号的回调函数
    (void *)base                // 回调函数的参数
    );

if (!signal_event || event_add(signal_event, NULL)<0) {     // 将事件处理器添加到 event_base 的事件处理器注册队列中
    fprintf(stderr, "Could not create/add a signal event!\n");
    return 1;
}
```

4、调用 `event_base_dispatch` 函数进入事件循环

``` cpp
event_base_dispatch(base);          // 进入事件循环
```

5、对象的释放

``` cpp
evconnlistener_free(listener);
event_free(signal_event);
event_base_free(base);
```

6、在创建 `evconnlistener` 对象时传入了一个回调函数：`listener_cb`

``` cpp
static void
listener_cb(struct evconnlistener *listener, evutil_socket_t fd,
    struct sockaddr *sa, int socklen, void *user_data)
{
    struct event_base *base = user_data;
    struct bufferevent *bev;                // 缓冲区

    /* 基于套接字创建一个缓冲区（套接字接收到数据之后存放在缓冲区中，套接字在发送数据之前先将数据存放在缓冲区中） */
    bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
    if (!bev) {
        fprintf(stderr, "Error constructing bufferevent!");
        event_base_loopbreak(base);
        return;
    }

    /* 设置回调函数 */
    bufferevent_setcb(
        bev,                // 缓冲区
        NULL,               // 读事件回调函数（设置为空）
        conn_writecb,       // 写事件回调函数  typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void *ctx);
        conn_eventcb,       // 事件处理回调函数  typedef void (*bufferevent_event_cb)(struct bufferevent *bev, short what, void *ctx);
        NULL                // 回调函数的最后一个参数
        );

    bufferevent_enable(bev, EV_WRITE);      // 启用缓存区的写功能
    bufferevent_disable(bev, EV_READ);      // 禁用缓冲区的读功能

    /* 将要发送的数据写入到该缓冲区 */
    bufferevent_write(bev, MESSAGE, strlen(MESSAGE));
}
```

7、 回调函数 `listener_cb` 又注册了写事件回调函数 `conn_writecb` 和 事件处理回调函数 `conn_eventcb`

``` cpp
static void
conn_writecb(struct bufferevent *bev, void *user_data)
{
    struct evbuffer *output = bufferevent_get_output(bev);      // 发送缓冲器中的数据
    if (evbuffer_get_length(output) == 0) {                     // 判断数据是否已经发送成功 为 0 代表发送成功
        printf("flushed answer\n");
        bufferevent_free(bev);                                  // 释放缓冲区
    }
}

static void
conn_eventcb(struct bufferevent *bev, short events, void *user_data)
{
    if (events & BEV_EVENT_EOF) {                           // 客户端关闭
        printf("Connection closed.\n");
    } else if (events & BEV_EVENT_ERROR) {
        printf("Got an error on the connection: %s\n",      // 链接产生错误
            strerror(errno));/*XXX win32*/
    }
    /* None of the other events can happen here, since we haven't enabled
     * timeouts */
    bufferevent_free(bev);
}
```

8、 `evsignal_new` 函数注册了信号处理函数 `signal_cb`

``` cpp
static void
signal_cb(evutil_socket_t sig, short events, void *user_data)
{
    struct event_base *base = user_data;
    struct timeval delay = {2, 0};

    printf("Caught an interrupt signal; exiting cleanly in two seconds.\n");

    event_base_loopexit(base, &delay);              // 退出事件循环
}
```

### 整体逻辑梳理

对于这个 `Demo`，整体的逻辑如下：

1. 新建一个 `event_base` ，然后在其上绑定一个 `listener` 来监听特定端口 `9995`;

2. 新建一个处理信号的事件 `signal_event`，并与上一步中的 `event_base` 绑定；

3. 调用 `event_base_dispatch` 来监控两个事件，包括指定的 `TCP` 连接及 `SIGINT` 信号;

4. 当监听到一个连接时，`libevent` 会触发 `listener_cb` 函数，该函数首先基于底层的 `socket` 建立一个 `bufferevent` ，并设置为可写不可读，最后调用 `bufferevent_write` 函数向其中缓存区写;

5. `bufferevent_write` 函数写后回触发写回调函数 `conn_writecb`，由于该例子中不需要其他操作，所以 `conn_writecb` 函数直接释放这个连接;

6. 当中断信号 `Ctrl+C` 出现时，`libevent` 会触发 `signal_event` 的回调函数 `signal_cb`，该函数会在 2 秒后停止 `event_base` 的监听，并退回到主函数；

7. 主函数依次释放 `listener`，`signal_event` 和 `event_base` 的资源，结束。

### 运行演示

1. 服务器端编译并运行

    ``` bash
    [root@VM_0_6_centos sample]# ./hello-world
    ```

2. 再开一个终端执行:

    ``` bash
    [root@VM_0_6_centos ~]# ncat 127.0.0.1 9995
    Hello, World!
    ```

3. 服务端显示:

    ``` bash
    [root@VM_0_6_centos sample]# ./hello-world

    ```

## 参考资料

### 源码

1. Github: <https://github.com/libevent/libevent>
2. 官方网站: <https://libevent.org/>

### 参考文档

1. `libevent` 源码深度剖析: <https://leanote.com/api/file/getAttach?fileId=577e1ad9ab644133ed002c8a>
2. Programming with Libevent: <http://www.wangafu.net/~nickm/libevent-book/>

### 参考博文

1. `libevent` 从入门到掌握 - 飞翔的猪: <https://zhuanlan.zhihu.com/p/87562010>
2. `libevent` 笔记 2：Hello_World - 孙敏铭: <https://www.cnblogs.com/sunminming/p/11862914.html>
