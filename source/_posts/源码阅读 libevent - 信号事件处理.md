---
title: 源码阅读 libevent - 信号事件处理
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 89de67d
date: 2020-11-01 15:37:32
---
不管使用的是什么多路 `IO` 复用模型，这些复用模型本身都是只支持读写 `IO` 事件的，而 `libevent` 支持的信号事件处理，就必须单独开发一套信号事件的处理逻辑，或者用某种办法将信号事件转换为 `IO` 事件进行处理。而 `libevent` 选择了后者，即统一事件源的方式。

<!--more-->

## 统一事件源的原理

统一事件源的工作原理如下：假如用户要监听 `SIGINT` 信号，那么在实现的内部就对 `SIGINT` 这个信号设置捕抓函数。此外，在实现的内部还要建立一条管道 (`pipe`)，并把这个管道加入到多路 `IO` 复用函数中。当 `SIGINT` 这个信号发生后，捕抓函数将会被调用。而这个捕抓函数的工作就是往管道写入一个字符 (这个字符往往等于所捕抓到信号的信号值)。此时，这个管道就变成是可读的了，多路 `IO` 复用函数能检测到这个管道变成可读的了。换言之，多路 `IO` 复用函数检测到 `SIGINT` 信号的发生，也就完成了对信号的监听工作。这个过程如下图所示：

``` mermaid
graph LR
S1((信号发生)) -- 触发 --> S2[信号捕捉函数]
S2 -- 写入 1 字节 --> P1[>>> 管道 >>>]
P1 -- 管道可读 --> IO{多路 IO 复用函数}
IO -- 监听到可读 --> F[用户处理函数]
```

## libevent 实现原理

按照上述统一事件源的原理介绍，`libevent` 内部实现的工作有：

1. 创建一个管道 `pipe`
2. 为这个 `pipe` 的一个读端创建一个 `event`，并将之加入到多路 `IO` 复用函数的监听之中
3. 设置信号捕捉函数
4. 有信号发生，就往 `pipe` 写入一个字节

统一事件源能够工作的一个原因是：多路 `IO` 复用函数都是可中断的。即处理完信号后，会从多路 `IO` 复用函数中退出，并将 `errno` 赋值为 `EINTR`。有些 `OS` 的某些系统调用，比如 `Linux` 的 `read`，即使被信号终端了，还是会自启动的。即不会从 `read` 函数中退出来。

下面开始介绍 `libevent` 统一事件源的实现代码。

## 关键结构体

``` cpp
/* file: event-internal.h */
struct event_base {
    const struct eventop *evsigsel;
    struct evsig_info sig;
    ...
    struct event_signal_map sigmap;
    ...
};

/* file: evsignal-internal.h */
typedef void (*ev_sighandler_t)(int);

struct evsig_info {
    struct event ev_signal; /* 用于监听 socketpair 读端的 event. ev_signal_pair[1] 为读端 */
    evutil_socket_t ev_signal_pair[2]; /* Socketpair used to send notifications from the signal handler */
    int ev_signal_added; /* True 标志已经将 ev_signal 这个 event 加入到 event_base 中了 */
    int ev_n_signals_added; /* 用户一共要监听多少个信号 */
    /* 数组。用户可能已经设置过某个信号的信号捕抓函数。但 libevent 还是要为这个信号设置另外一个信号捕抓函数，
    ** 此时，就要保存用户之前设置的信号捕抓函数。当用户不要监听这个信号时，就能够恢复用户之前的捕抓函数。
    ** 因为是有多个信号，所以得用一个数组保存。 */
#ifdef EVENT__HAVE_SIGACTION
    struct sigaction **sh_old;
#else
    ev_sighandler_t **sh_old;
#endif
    int sh_old_max; /* 数组的长度. */
};
```

## 初始化

``` cpp
int evsig_init_(struct event_base *base) {
    if (evutil_make_internal_pipe_(base->sig.ev_signal_pair) == -1) {
        event_sock_err(1, -1, "%s: socketpair", __func__);
        return -1;
    }

    if (base->sig.sh_old) mm_free(base->sig.sh_old);
    base->sig.sh_old = NULL;
    base->sig.sh_old_max = 0;

    event_assign(&base->sig.ev_signal, base, base->sig.ev_signal_pair[0], EV_READ | EV_PERSIST, evsig_cb, base);
    base->sig.ev_signal.ev_flags |= EVLIST_INTERNAL;
    event_priority_set(&base->sig.ev_signal, 0);
    base->evsigsel = &evsigops;

    return 0;
}
```

### 创建管道

`libevent` 中该管道的实现方式有三种方式，会根据编译期系统支持的函数确定，三种方式依次为：

1. `pipe2`：若支持，则 `libevent` 定义宏 `EVENT__HAVE_PIPE2`
2. `pipe`：若支持 则 `libevent` 定义宏 `EVENT__HAVE_PIPE`
3. `socketpair`：以上两种方式不支持时使用。

三种方式均会创建一个文件描述符数组，数组中有两个元素：`fd[0]` 和 `fd[1]`，程序从 `fd[1]` 端写入数据，从 `fd[0]` 端读取数据。

三种方式的实现均在以下代码中：

``` cpp
int evutil_make_internal_pipe_(evutil_socket_t fd[2]) {
#if defined(EVENT__HAVE_PIPE2)
    if (pipe2(fd, O_NONBLOCK|O_CLOEXEC) == 0) return 0;
#if defined(EVENT__HAVE_PIPE)
    if (pipe(fd) == 0) {
        if (evutil_fast_socket_nonblocking(fd[0]) < 0 || evutil_fast_socket_nonblocking(fd[1]) < 0 ||
            evutil_fast_socket_closeonexec(fd[0]) < 0 || evutil_fast_socket_closeonexec(fd[1]) < 0) {
            close(fd[0]);
            close(fd[1]);
            fd[0] = fd[1] = -1;
            return -1;
        }
        return 0;
    } else event_warn("%s: pipe", __func__);
#endif

#define LOCAL_SOCKETPAIR_AF AF_UNIX
    if (evutil_socketpair(LOCAL_SOCKETPAIR_AF, SOCK_STREAM, 0, fd) == 0) {
        if (evutil_fast_socket_nonblocking(fd[0]) < 0 || evutil_fast_socket_nonblocking(fd[1]) < 0 ||
            evutil_fast_socket_closeonexec(fd[0]) < 0 || evutil_fast_socket_closeonexec(fd[1]) < 0) {
            evutil_closesocket(fd[0]);
            evutil_closesocket(fd[1]);
            fd[0] = fd[1] = -1;
            return -1;
        }
        return 0;
    }
    fd[0] = fd[1] = -1;
    return -1;
}
```

通过 `pipe2` 方式创建的管道代码最为简单，而通过 `pipe` 和 `socketpair` 创建的管道涉及到 `4` 个 `libevent` 封装的函数 `evutil_socketpair`/`evutil_fast_socket_nonblocking`/`evutil_fast_socket_closeonexec`/`evutil_closesocket` ，我们依次进行分析。

#### evutil_socketpair

``` cpp
int evutil_socketpair(int family, int type, int protocol, evutil_socket_t fd[2]) {
    return evutil_ersatz_socketpair_(family, type, protocol, fd);
}

int evutil_ersatz_socketpair_(int family, int type, int protocol, evutil_socket_t fd[2]) {
#define ERR(e) e
    evutil_socket_t listener = -1;
    evutil_socket_t connector = -1;
    evutil_socket_t acceptor = -1;
    struct sockaddr_in listen_addr;
    struct sockaddr_in connect_addr;
    ev_socklen_t size;
    int saved_errno = -1;
    int family_test;

    family_test = family != AF_INET;
#ifdef AF_UNIX
    family_test = family_test && (family != AF_UNIX);
#endif
    if (protocol || family_test) {
        EVUTIL_SET_SOCKET_ERROR(ERR(EAFNOSUPPORT));
        return -1;
    }

    if (!fd) {
        EVUTIL_SET_SOCKET_ERROR(ERR(EINVAL));
        return -1;
    }

    listener = socket(AF_INET, type, 0);
    if (listener < 0) return -1;
    memset(&listen_addr, 0, sizeof(listen_addr));
    listen_addr.sin_family = AF_INET;
    listen_addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    listen_addr.sin_port = 0;   /* kernel chooses port.  */
    if (bind(listener, (struct sockaddr *) &listen_addr, sizeof (listen_addr)) == -1) goto tidy_up_and_fail;
    if (listen(listener, 1) == -1) goto tidy_up_and_fail;

    connector = socket(AF_INET, type, 0);
    if (connector < 0) goto tidy_up_and_fail;

    memset(&connect_addr, 0, sizeof(connect_addr));

    /* We want to find out the port number to connect to.  */
    size = sizeof(connect_addr);
    if (getsockname(listener, (struct sockaddr *) &connect_addr, &size) == -1) goto tidy_up_and_fail;
    if (size != sizeof (connect_addr)) goto abort_tidy_up_and_fail;
    if (connect(connector, (struct sockaddr *) &connect_addr, sizeof(connect_addr)) == -1) goto tidy_up_and_fail;

    size = sizeof(listen_addr);
    acceptor = accept(listener, (struct sockaddr *) &listen_addr, &size);
    if (acceptor < 0) goto tidy_up_and_fail;
    if (size != sizeof(listen_addr)) goto abort_tidy_up_and_fail;
    /* Now check we are talking to ourself by matching port and host on the
       two sockets.  */
    if (getsockname(connector, (struct sockaddr *) &connect_addr, &size) == -1) goto tidy_up_and_fail;
    if (size != sizeof (connect_addr) || listen_addr.sin_family != connect_addr.sin_family
        || listen_addr.sin_addr.s_addr != connect_addr.sin_addr.s_addr || listen_addr.sin_port != connect_addr.sin_port)
        goto abort_tidy_up_and_fail;
    evutil_closesocket(listener);
    fd[0] = connector;
    fd[1] = acceptor;

    return 0;

 abort_tidy_up_and_fail:
    saved_errno = ERR(ECONNABORTED);
 tidy_up_and_fail:
    if (saved_errno < 0) saved_errno = EVUTIL_SOCKET_ERROR();
    if (listener != -1) evutil_closesocket(listener);
    if (connector != -1) evutil_closesocket(connector);
    if (acceptor != -1) evutil_closesocket(acceptor);

    EVUTIL_SET_SOCKET_ERROR(saved_errno);
    return -1;
#undef ERR
}
```

从代码中可以看出，创建 `socketpair` 需要进行以下操作：

先创建一个 `listener`，监听本地环回端口，相当于服务端。创建一个 `connector` 作为客户端向 `listener` 发起连接，`listener` 通过 `accept` 函数与 `connecotr` 建立连接，新连接套接字为 `acceptor`，此时 `acceptor` 和 `connector` 就可以互发消息，成为一对全双工 `socketpair`，流程图如下所示

``` mermaid
sequenceDiagram
participant connector
participant listener
Note over listener : socket, bind, listen
Note over connector : socket, getsockname
connector ->> listener : connect
Note over listener : accept
activate listener
listener -->> connector : send
Note over connector : select, poll, epoll
deactivate listener
```

#### 其他三个函数

``` cpp
static int evutil_fast_socket_nonblocking(evutil_socket_t fd) {
    if (fcntl(fd, F_SETFL, O_NONBLOCK) == -1) {
        event_warn("fcntl(%d, F_SETFL)", fd);
        return -1;
    }
    return 0;
}

static int evutil_fast_socket_closeonexec(evutil_socket_t fd) {
    if (fcntl(fd, F_SETFD, FD_CLOEXEC) == -1) {
        event_warn("fcntl(%d, F_SETFD)", fd);
        return -1;
    }
    return 0;
}

int evutil_closesocket(evutil_socket_t sock) {
    return close(sock);
}
```

这三个函数对应三个功能：

1. 将套接字设置为非阻塞属性
    - 如果 `recv` 调用没有可读取的数据, 或者如果 `send` 操作将阻塞, `recv` 或 `send` 调用返回 `-1` 和 `EAGAIN` 错误
2. 将套接字设置为在 exec 时 关闭：
    - `FD_CLOEXEC` 表示当程序执行 `exec` 函数时本 `fd` 将被系统自动关闭, 表示不传递给 `exec` 创建的新进程, 如果设置为 `fcntl(fd, F_SETFD, 0);` 那么 `fd` 将保持打开状态复制到 `exec` 创建的新进程中
3. 关闭套接字

### 初始化事件结构体

调用 `event_assign()` 函数，设置 `ev_signal` 的监听对象就是这对套接字之一，并且监听读事件，回调函数为 `evsig_cb`。有关回调函数的细节在 [信号事件的处理](# 信号事件的处理) 小节讲解，最后就是将信号处理后端函数结构体绑定到 `event_base` 中的 `evsigsel` 上。

> 有关 `event_assign()` 的分析参见：{% post_link "源码阅读 libevent - 结构体：event" %}

## 创建信号 event

创建一个 `io event` 的函数是 `event_new()`，如果要创建信号 `event`，一种方法是直接设置 `event_new()` 的参数为 `EV_SIGNAL`，不过这种方法对于用户来说是非常不友好的。为此，`libevent` 在 `event.h` 中进行了如下的宏定义：

``` cpp
#define evsignal_add(ev, tv)                event_add((ev), (tv))
#define evsignal_assign(ev, b, x, cb, arg)  event_assign((ev), (b), (x), EV_SIGNAL|EV_PERSIST, cb, (arg))
#define evsignal_new(b, x, cb, arg)         event_new((b), (x), EV_SIGNAL|EV_PERSIST, (cb), (arg))
#define evsignal_del(ev)                    event_del(ev)
#define evsignal_pending(ev, tv)            event_pending((ev), EV_SIGNAL, (tv))
#define evsignal_initialized(ev)            event_initialized(ev)
```

可以看到，这里实际上对 `io event` 的相关函数的参数进行了特殊化处理，最终得到了信号 `event` 的相关函数。此时，就可以通过 `evsignal_new` 函数来创建一个信号 `event` 了，实际上就是创建了一个 `EV_SIGNAL|EV_PERSIST` 的永久信号事件。而该函数内部实际上又会调用 `event_assign` 函数。需要注意的是，创建普通 `event` 时第二个参数传入的是需要监听的的文件描述符，而这里创建信号 `event` 时传入的第二个参数则应当是需要监听的信号值了，比如说需要监听的信号是 `SIGUSR1`，那么调用 `evsignal_new` 时，传入的第二个参数就应该直接使用 `SIGUSR1`。`evsignal_new` 的第三个参数 `cb` 自然就应当是用户需要监听的信号发生后，期待调用的函数。

## 添加信号 event

如上所述，添加一个信号 `event` 使用 `evsignal_add()` 函数，实际上就是 `event_add()` 函数。前面创建的信号 `event`，其 `events` 成员已经被设置为了 `EV_SIGNAL|EV_PERSIST`。因此，`event_add()` 函数中会调用 `evmap_signal_add()` 函数将该 `event` 添加到 `event_signal_map` 中。

> `event_add()` 相关说明参见：{% post_link "源码阅读 libevent - 结构体：event" %}
> `evmap_signal_add()` 和 `event_signal_map` 相关说明参见：{% post_link "源码阅读 libevent - event_signal_map" %}

### evmap_signal_add

``` cpp
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

`evmap_signal_add()` 除了将该 `event` 添加到 `event_signal_map` 中，还调用了 `evsel->add` 回调函数，其真正的实现函数为 `evsig_add()`

### evsig_add

``` cpp
static int evsig_add(struct event_base *base, evutil_socket_t evsignal, short old, short events, void *p) {
    struct evsig_info *sig = &base->sig;
    EVUTIL_ASSERT(evsignal>= 0 && evsignal < NSIG);
    evsig_base = base;
    evsig_base_n_signals_added = ++sig->ev_n_signals_added;
    evsig_base_fd = base->sig.ev_signal_pair[1];

    if (evsig_set_handler_(base, (int)evsignal, evsig_handler) == -1) goto err;

    if (!sig->ev_signal_added) {
        if (event_add_nolock_(&sig->ev_signal, NULL, 0)) goto err;
        sig->ev_signal_added = 1;
    }
    return (0);
err:
    --evsig_base_n_signals_added;
    --sig->ev_n_signals_added;
    return (-1);
}
```

从后面的那个 `if` 语句可以得知，当 `sig->ev_signal_added` 变量为 `0` 时 (即用户第一次监听一个信号)，就会将 `ev_signal` 这个 `event` 加入到 `event_base` 中。从前面的统一事件源可以得知，这个 `ev_signal` 的作用就是通知 `event_base`，有信号发生了。只需一个 `event` 即可完成工作，即使用户要监听多个不同的信号，因为这个 `event` 已经和 `socketpair` 的读端相关联了。如果要监听多个信号，那么就在信号处理函数中往这个 `socketpair` 写入不同的值即可。`event_base` 能监听到可读，并可以从读到的内容可以判断是哪个信号发生了。

从代码中也可得知，`libevent` 并不会为每一个信号监听创建一个 `event`。它只会创建一个全局的专门用于监听信号的 `event`。这个也是统一事件源的工作原理。

### evsig_set_handler_

`evsig_add()` 函数还调用了 `_evsig_set_handler()` 函数完成设置 `libevent` 内部的信号捕抓函数。

``` cpp
int evsig_set_handler_(struct event_base *base, int evsignal, void (__cdecl *handler)(int)) {
    struct sigaction sa;
    struct evsig_info *sig = &base->sig;

    if (evsignal>= sig->sh_old_max) {
        int new_max = evsignal + 1;
        event_debug(("%s: evsignal (%d) >= sh_old_max (%d), resizing", __func__, evsignal, sig->sh_old_max));
        p = mm_realloc(sig->sh_old, new_max * sizeof(*sig->sh_old));
        if (p == NULL) return (-1);
        memset((char *)p + sig->sh_old_max * sizeof(*sig->sh_old), 0, (new_max - sig->sh_old_max) * sizeof(*sig->sh_old));
        sig->sh_old_max = new_max;
        sig->sh_old = p;
    }

    /* allocate space for previous handler out of dynamic array */
    sig->sh_old[evsignal] = mm_malloc(sizeof *sig->sh_old[evsignal]);
    if (sig->sh_old[evsignal] == NULL) {
        event_warn("malloc");
        return (-1);
    }

    /* save previous handler and setup new handler */
    memset(&sa, 0, sizeof(sa));
    sa.sa_handler = handler;
    sa.sa_flags |= SA_RESTART;
    sigfillset(&sa.sa_mask);

    if (sigaction(evsignal, &sa, sig->sh_old[evsignal]) == -1) {
        event_warn("sigaction");
        mm_free(sig->sh_old[evsignal]);
        sig->sh_old[evsignal] = NULL;
        return (-1);
    }

    return (0);
}
```

### evsig_handler

`_evsig_set_handler()` 函数设置 `libevent` 内部的信号捕抓函数为 `evsig_handler()`

``` cpp
static void __cdecl evsig_handler(int sig) {
    int save_errno = errno;
    ev_uint8_t msg;
    if (evsig_base == NULL) return;
    /* Wake up our notification mechanism */
    msg = sig;
    int r = write(evsig_base_fd, (char*)&msg, 1);
    errno = save_errno;
}
```

主要功能是发送一字节到管道或 `socket`。

## 信号 event 的处理

### 回调函数 evsig_cb

``` cpp
/* Callback for when the signal handler write a byte to our signaling socket */
static void evsig_cb(evutil_socket_t fd, short what, void *arg) {
    static char signals[1024];
    ev_ssize_t n;
    int i;
    int ncaught[NSIG];
    struct event_base *base;

    base = arg;

    memset(&ncaught, 0, sizeof(ncaught));

    while (1) {
        n = read(fd, signals, sizeof(signals));
        if (n == -1) {
            int err = evutil_socket_geterror(fd);
            if (! EVUTIL_ERR_RW_RETRIABLE(err)) event_sock_err(1, fd, "%s: recv", __func__);
            break;
        } else if (n == 0) break;
        for (i = 0; i < n; ++i) {
            ev_uint8_t sig = signals[i];
            if (sig < NSIG) ncaught[sig]++;
        }
    }

    for (i = 0; i < NSIG; ++i)
        if (ncaught[i]) evmap_signal_active_(base, i, ncaught[i]);
}
```

`evsig_cb()` 会从读端套接字中读取数据到 `signals` 数组中，读出的每一个字节都代表一个发生的信号值，用 `ncaught` 数组来存储每个信号发生的次数，并且对于每个发生的信号，都调用 `evmap_signal_active` 进行处理：

### evmap_signal_active_

`evmap_signal_active` 主要是找到 `event_signal_map` 中，监听信号值为 `sig` 的那一个 `evmap_signal`，在这个 `evmap_signal` 中，包含了所有监听信号值为 `sig` 的 `event` 组成的双向链表，然后直接遍历这个双向链表，把每个元素都按照 `EV_SIGNAL` 的激活方式调用 `event_active_nolock` 函数。

``` cpp
void evmap_signal_active_(struct event_base *base, evutil_socket_t sig, int ncalls) {
    struct event_signal_map *map = &base->sigmap;
    struct evmap_signal *ctx;
    struct event *ev;

    if (sig < 0 || sig>= map->nentries) return;
    GET_SIGNAL_SLOT(ctx, map, sig, evmap_signal);
    if (!ctx) return;
    LIST_FOREACH(ev, &ctx->events, ev_signal_next)
    event_active_nolock_(ev, EV_SIGNAL, ncalls);
}
```

对于 `event_active_nolock` 函数来说，主要任务是调用 `event_callback_activate_nolock_` 把传入的 `event` 添加到激活队列中，如下所示：

``` cpp
void event_active_nolock_(struct event *ev, int res, short ncalls) {
    ......
    event_callback_activate_nolock_(base, event_to_event_callback(ev));
}

int event_callback_activate_nolock_(struct event_base *base, struct event_callback *evcb) {
    ......
    event_queue_insert_active(base, evcb);
    if (EVBASE_NEED_NOTIFY(base)) evthread_notify_base(base);
    return r;
}
```

到这里，用来监听用户指定的信号值的那个信号 `event` 就被添加到了激活队列中，接下来，就等待主循环 `event_base_loop()` 处理激活队列时去处理那个信号 `event`。

### event_signal_closure

激活处理的过程就是 `event_base_loop()` --> `event_process_active()` --> `event_process_active_single_queue()`，在 `event_process_active_single_queue` 函数中，会判断激活处理事件的回调关闭方式 `ev_closure`，而对于信号 `event` 来说，在 `evsignal_new()` 时由于传入的参数为 `EV_SIGNAL|EV_PERSIST`，因此 `evsignal_new()` 内部调用的 `event_assign()` 会直接设置信号 `event` 的 `ev_closure` 为 `EV_CLOSURE_SIGNAL`，也就是说，处理激活的信号 `event` 最终是通过 `event_signal_closure()` 函数实现的，该函数定义如下：

``` cpp
/* "closure" function called when processing active signal events */
static inline void event_signal_closure(struct event_base *base, struct event *ev) {
    short ncalls;
    int should_break;
    ncalls = ev->ev_ncalls;
    if (ncalls != 0) ev->ev_pncalls = &ncalls;
    EVBASE_RELEASE_LOCK(base, th_base_lock);
    while (ncalls) {
        ncalls--;
        ev->ev_ncalls = ncalls;
        if (ncalls == 0) ev->ev_pncalls = NULL;
        (*ev->ev_callback)(ev->ev_fd, ev->ev_res, ev->ev_arg);

        EVBASE_ACQUIRE_LOCK(base, th_base_lock);
        should_break = base->event_break;
        EVBASE_RELEASE_LOCK(base, th_base_lock);

        if (should_break) {
            if (ncalls != 0) ev->ev_pncalls = NULL;
            return;
        }
    }
}
```

这里的 `ncalls`，实际上就是 `evsig_cb` 函数中记录的信号值 `sig` 的发生次数，信号 `event` 激活时只处理一次，但是在这一次中会根据信号发生的次数来决定调用多少次回调函数，而这里的回调函数就是用户最开始设定的当信号发生时应当调用的函数了。

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [libevent-src-analysis/libevent 源码分析. md at master · KelvinYin/libevent-src-analysis](https://github.com/KelvinYin/libevent-src-analysis/blob/master/libevent%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
