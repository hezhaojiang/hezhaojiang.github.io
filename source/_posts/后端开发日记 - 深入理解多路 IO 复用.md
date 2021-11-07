---
title: 后端开发日记 - 深入理解多路 IO 复用
categories:
  - 后端开发日记
tags:
  - 网络编程
abbrlink: 1a0f9f15
date: 2020-06-01 16:27:00
---

## Linux I/O 模型

* `Blocking IO` - 阻塞 IO
* `NoneBlocking IO` - 非阻塞 IO
* `IO multiplexing` - IO 多路复用
* `signal driven IO` - 信号驱动 IO
* `asynchronous IO` - 异步 IO

`Select`, `Poll`, `Epoll` 就是 `IO multiplexing` 的三种机制。

<!-- more -->

## Select

### Select 函数声明

首先通过 `man` 命令查看下 `select` 的函数声明：

```c++
#include <sys/select.h>

int select(int nfds, fd_set *restrict readfds,
    fd_set *restrict writefds, fd_set *restrict errorfds,
    struct timeval *restrict timeout);

void FD_CLR(int fd, fd_set *fdset);
int  FD_ISSET(int fd, fd_set *fdset);
void FD_SET(int fd, fd_set *fdset);
void FD_ZERO(fd_set *fdset);
```

### Select 函数参数说明

`int nfds` : `nfds` 指定待测试的文件描述字个数，它的值是待测试的最大描述字加 `1` 。

> The nfds argument specifies the range of descriptors to be tested. The first nfds descriptors shall be checked in each set; that is, the descriptors from zero through nfds-1 in the descriptor sets shall be examined.

`fd_set *readset` , `fd_set *writeset` , `fd_set *exceptset` : `fd_set` 可以理解为一个集合，这个集合中存放的是文件描述符( `file descriptor` )。这三个参数指定我们要让内核测试读、写和异常条件的文件描述符集合。如果对某一个的条件不感兴趣，就可以把它设为空指针。

> If the readfds argument is not a null pointer, it points to an object of type fd_set that on input specifies the file descriptors to be checked for being ready to read, and on output indicates which file descriptors are ready to read.
> If the writefds argument is not a null pointer, it points to an object of type fd_set that on input specifies the file descriptors to be checked for being ready to write, and on output indicates which file descriptors are ready to write.
> If the errorfds argument is not a null pointer, it points to an object of type fd_set that on input specifies the file descriptors to be checked for error conditions pending, and on output indicates which file descriptors have error conditions pending.

`const struct timeval *timeout` : `timeout` 告知内核等待所指定文件描述符集合中的任何一个就绪可花多少时间。其 `timeval` 结构用于指定这段时间的秒数和微秒数。

> The timeout parameter controls how long the pselect() or select() function shall take before timing out. If the timeout parameter is not a null pointer, it specifies a maximum interval to wait for the selection to complete. If the specified time interval expires without any requested operation becoming ready, the function shall return. If the timeout parameter is a null pointer, then the call to pselect() or select() shall block indefinitely until at least one descriptor meets the specified criteria. To effect a poll, the timeout parameter should not be a null pointer, and should point to a zero-valued timespec structure.
> If the readfds, writefds, and errorfds arguments are all null pointers and the timeout argument is not a null pointer, the pselect() or select() function shall block for the time specified, or until interrupted by a signal. If the readfds, writefds, and errorfds arguments are all null pointers and the timeout argument is a null pointer, the pselect() or select() function shall block until interrupted by a signal.

### Select 函数返回值

`select` 的返回类型为 `int` , 若有就绪描述符返回其数目，若超时则为 `0` ，若出错则为 `-1` 。

> Upon successful completion, the pselect() and select() functions shall return the total number of bits set in the bit masks. Otherwise, -1 shall be returned, and errno shall be set to indicate the error.

`FD_CLR(), FD_SET(), FD_ZERO()` 无返回值, `FD_ISSET()` 在当前文件描述符被 `FD_SET()` 时会返回非 `0` 值, 否则返回 `0` 。

> FD_CLR(), FD_SET(), and FD_ZERO() do not return a value. FD_ISSET() shall return a non-zero value if the bit for the file descriptor fd is set in the file descriptor set pointed to by fdset, and 0 otherwise.

### Select 实现框架

```c++
fd_set read_fd;
struct timeval tv;
int err;

while (1) {
    FD_ZERO(&read_fd);
    FD_SET(0,&read_fd);

    tv.tv_sec = 5;
    tv.tv_usec = 0;

    err = select(1,&read_fd,NULL,NULL,&tv);

    if (err == 0) {
        printf("select time out!\n");
    }
    else if (err == -1) {
        printf("fail to select!\n");
    }
    else {
        printf("data is available!\n");
    }
}
```

1. 执行 `fd_set set;`  `FD_ZERO(&set);` 则 `set` 用位表示是 `0000,0000`
2. 若 `fd = 5` , 执行 `FD_SET(fd, &set);` 后 `set` 变为 `0001,0000` (第 `5` 位置为 `1` )

3. 若再加入 `fd=2` ， `fd=1` , 则 `set` 变为 `0001,0011`
4. 执行 `select(6, &set, 0, 0, 0)` 阻塞等待

5. 若 `fd = 1` , `fd = 2` 上都发生可读事件，则 `select` 返回，此时 `set` 变为 `0000,0011` 。注意：没有事件发生的 `fd = 5` 被清空

### Select 的优缺点

1. 被监控的fds集合限制为 `1024` ， `1024` 太小了，我们希望能够有个比较大的可监控 `fds` 集合。

2. `fds` 集合需要从用户空间拷贝到内核空间的问题，我们希望不需要拷贝。

3. 当被监控的 `fds` 中某些有数据可读的时候，我们希望通知更加精细一点，就是我们希望能够从通知中得到有可读事件的 `fds` 列表，而不是需要遍历整个 `fds` 来收集。

## Epoll

### Epoll 函数集声明

``` c++
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

### Epoll 函数集参数说明

### Epoll 函数集返回值

### Epoll实现框架

``` c++
#define MAX_EVENTS 10
struct epoll_event ev, events[MAX_EVENTS];
int listen_sock, conn_sock, nfds, epollfd;

/* Set up listening socket, 'listen_sock' (socket(),
    bind(), listen()) */

epollfd = epoll_create(10);
if (epollfd == -1) {
    perror("epoll_create");
    exit(EXIT_FAILURE);
}

ev.events = EPOLLIN;
ev.data.fd = listen_sock;
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
    perror("epoll_ctl: listen_sock");
    exit(EXIT_FAILURE);
}

for (;;) {
    nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
    if (nfds == -1) {
        perror("epoll_wait");
        exit(EXIT_FAILURE);
    }

    for (n = 0; n < nfds; ++n) {
        if (events[n].data.fd == listen_sock) {
            conn_sock = accept(listen_sock,
                            (struct sockaddr *) &local, &addrlen);
            if (conn_sock == -1) {
                perror("accept");
                exit(EXIT_FAILURE);
            }
            setnonblocking(conn_sock);
            ev.events = EPOLLIN | EPOLLET;
            ev.data.fd = conn_sock;
            if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                        &ev) == -1) {
                perror("epoll_ctl: conn_sock");
                exit(EXIT_FAILURE);
            }
        } else {
            do_use_fd(events[n].data.fd);
        }
    }
}
```

### Epoll 的优缺点
