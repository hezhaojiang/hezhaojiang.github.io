---
title: 源码阅读 libevent - 多线程：锁和条件变量
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 52b3bda8
date: 2020-10-21 14:53:56
---
`libevent` 的多线程接口文件相比日志模块和内存管理模块文件较多，可大致分为三部分：

1. 内部接口所在头文件：`evthread-internal.h`
2. 外部接口所在头文件：`thread.h`
3. 接口实现所在文件：`evthread.c` & `evthread_pthread.c` & `evthread_win32.c`

<!--more-->

## 开启多线程

`libevent` 的多线程模块与日志模块和内存管理模块不同，在默认的情况下并没有进行实现，而是仅仅提供了函数接口，即 `libevent` 在默认情况下是不支持多线程的。

如果要让 `libevent` 支持多线程，则需要在使用 `libevent` 之前，即在调用 `event_base_new` 函数之前调用 `evthread_use_windows_threads()` 或 `evthread_use_pthreads()` 或 `evthread_set_lock_callbacks` 函数定制自己的多线程、锁、条件变量才会开启多线程功能。

以上三个线程定制函数均定义在 `thread.h`，而线程的使用接口定义在 `evthread-internal.h`，即对于用户来说，只能定制 `libevent` 的线程函数，而这些函数的使用方式还是由 `libevent` 自行管理的。

## 锁和条件变量结构体

`libevent` 允许用户定制自己的锁和条件变量。其实现原理和日志和内存分配一样，都是内部有一个全局变量，全局变量中保存着自己定制的相关功能实现函数。而要定制自己的锁和条件变量，就是对这个全局变量进行赋值。

`libevent` 中锁和条件变量定制对应的全局变量如下：

``` cpp
struct evthread_lock_callbacks {
    int lock_api_version;
    unsigned supported_locktypes;
    void *(*alloc)(unsigned locktype);
    void (*free)(void *lock, unsigned locktype);
    int (*lock)(unsigned mode, void *lock);
    int (*unlock)(unsigned mode, void *lock);
};

struct evthread_condition_callbacks {
    int condition_api_version;
    void *(*alloc_condition)(unsigned condtype);
    void (*free_condition)(void *cond);
    int (*signal_condition)(void *cond, int broadcast);
    int (*wait_condition)(void *cond, void *lock, const struct timeval *timeout);
};

GLOBAL struct evthread_lock_callbacks evthread_lock_fns_ = {0, 0, NULL, NULL, NULL, NULL};
GLOBAL struct evthread_condition_callbacks evthread_cond_fns_ = {0, NULL, NULL, NULL, NULL};
```

该全局变量结构体中每个参数意义如下:

1. `evthread_lock_fns_`
    - `.lock_api_version` 锁函数的版本
    - `.supported_locktypes` 指示支持的锁类型
    - `.alloc` 初始化（申请）一个锁
    - `.free` 反初始化（销毁）一个锁
    - `.lock` 加锁
    - `.unlock` 解锁
2. `evthread_cond_fns_`
    - `.condition_api_version` 条件变量函数版本
    - `.alloc_condition` 初始化（申请）一个条件变量
    - `.free_condition` 反初始化（销毁）一个条件变量
    - `.signal_condition` 通知
    - `.wait_condition` 等待通知

## libevent 提供的多线程函数

对于用户来说，如果想让 `libevent` 支持多线程。`Windows` 用户直接调用 `evthread_use_windows_threads()`，遵循 `POSIX` 标准的系统直接调用 `evthread_use_pthreads()` 就可以了。

在 `Linux 2.6` 以上版本中，已经对 `pthread` 线程有了完善的支持，以下我们就分析一下 `libevent` 提供的多线程函数 `evthread_use_pthreads()`

``` cpp
#define EVTHREAD_LOCK_API_VERSION 1
#define EVTHREAD_LOCKTYPE_RECURSIVE 1
#define EVTHREAD_CONDITION_API_VERSION 1

static pthread_mutexattr_t attr_recursive;

int evthread_use_pthreads(void) {
    struct evthread_lock_callbacks cbs = {
        EVTHREAD_LOCK_API_VERSION,
        EVTHREAD_LOCKTYPE_RECURSIVE,
        evthread_posix_lock_alloc,
        evthread_posix_lock_free,
        evthread_posix_lock,
        evthread_posix_unlock
    };
    struct evthread_condition_callbacks cond_cbs = {
        EVTHREAD_CONDITION_API_VERSION,
        evthread_posix_cond_alloc,
        evthread_posix_cond_free,
        evthread_posix_cond_signal,
        evthread_posix_cond_wait
    };
    /* Set ourselves up to get recursive locks. */
    if (pthread_mutexattr_init(&attr_recursive)) return -1;
    if (pthread_mutexattr_settype(&attr_recursive, PTHREAD_MUTEX_RECURSIVE)) return -1;

    evthread_set_lock_callbacks(&cbs);
    evthread_set_condition_callbacks(&cond_cbs);
    evthread_set_id_callback(evthread_posix_get_id);
    return 0;
}
```

从代码可看出，`evthread_use_pthreads` 函数主要有三个步骤：

1. 定义并初始化了一个 `evthread_lock_callbacks` 结构体和一个 `evthread_condition_callbacks` 结构体，以便后续定制修改全局锁和条件变量结构体
2. 初始化全局变量 `attr_recursive`，这个变量会在初始化锁时被传递，代表需要申请锁的属性，在该函数里其被设置成为递归属性。
3. 调用全局锁和条件变量结构体的 `set` 函数将步骤 1 中的结构体赋值到全局变量中。

### 锁相关函数

#### evthread_posix_lock_alloc

``` cpp
static void *evthread_posix_lock_alloc(unsigned locktype) {
    pthread_mutexattr_t *attr = NULL;
    pthread_mutex_t *lock = mm_malloc(sizeof(pthread_mutex_t));
    if (!lock) return NULL;
    if (locktype & EVTHREAD_LOCKTYPE_RECURSIVE) attr = &attr_recursive;
    if (pthread_mutex_init(lock, attr)) {
        mm_free(lock);
        return NULL;
    }
    return lock;
}
```

`evthread_posix_lock_alloc` 函数封装了 `pthread_mutex_init` 函数，根据传入参数 `locktype` 不同，创建不同类型的锁：

- `locktype == 0` ：创建普通互斥锁
- `locktype == EVTHREAD_LOCKTYPE_RECURSIVE` ：创建递归锁
- `locktype == EVTHREAD_LOCKTYPE_READWRITE` ：创建读写锁（`evthread_posix_lock_alloc` 函数中未实现）

#### evthread_posix_lock_free

``` cpp
static void evthread_posix_lock_free(void *lock_, unsigned locktype) {
    pthread_mutex_t *lock = lock_;
    pthread_mutex_destroy(lock);
    mm_free(lock);
}
```

`evthread_posix_lock_free` 函数对 `pthread_mutex_destroy` 函数进行了封装。

#### evthread_posix_lock

``` cpp
static int evthread_posix_lock(unsigned mode, void *lock_) {
    pthread_mutex_t *lock = lock_;
    if (mode & EVTHREAD_TRY) return pthread_mutex_trylock(lock);
    else return pthread_mutex_lock(lock);
}
```

`evthread_posix_lock` 对 `pthread` 的 `lock` 相关函数进行了封装：

- 当 `mode == 0` 时，相当于调用 `pthread_mutex_lock` 函数
- 当 `mode == EVTHREAD_TRY` 时，相当于调用 `pthread_mutex_trylock` 函数

#### evthread_posix_unlock

``` cpp
static int evthread_posix_unlock(unsigned mode, void *lock_) {
    pthread_mutex_t *lock = lock_;
    return pthread_mutex_unlock(lock);
}
```

`evthread_posix_unlock` 函数对 `pthread_mutex_unlock` 函数进行了封装。

### 条件变量相关函数

#### evthread_posix_cond_alloc

``` cpp
static void * evthread_posix_cond_alloc(unsigned condflags) {
    pthread_cond_t *cond = mm_malloc(sizeof(pthread_cond_t));
    if (!cond) return NULL;
    if (pthread_cond_init(cond, NULL)) {
        mm_free(cond);
        return NULL;
    }
    return cond;
}
```

`evthread_posix_cond_alloc` 函数对 `pthread_cond_init` 函数进行了封装。

#### evthread_posix_cond_free

``` cpp
static void evthread_posix_cond_free(void *cond_) {
    pthread_cond_t *cond = cond_;
    pthread_cond_destroy(cond);
    mm_free(cond);
}
```

`evthread_posix_cond_free` 函数对 `pthread_cond_destroy` 函数进行了封装。

#### evthread_posix_cond_signal

``` cpp
static int evthread_posix_cond_signal(void *cond_, int broadcast) {
    pthread_cond_t *cond = cond_;
    int r;
    if (broadcast) r = pthread_cond_broadcast(cond);
    else r = pthread_cond_signal(cond);
    return r ? -1 : 0;
}
```

`evthread_posix_cond_signal` 函数对 `pthread` 的通知相关函数进行了封装：

- 当 `broadcast == 0` 时，相当于调用 `pthread_cond_signal` 函数
- 当 `broadcast != 0` 时，相当于调用 `pthread_cond_broadcast` 函数

#### evthread_posix_cond_wait

``` cpp
static int evthread_posix_cond_wait(void *cond_, void *lock_, const struct timeval *tv) {
    int r;
    pthread_cond_t *cond = cond_;
    pthread_mutex_t *lock = lock_;

    if (tv) {
        struct timeval now, abstime;
        struct timespec ts;
        evutil_gettimeofday(&now, NULL);
        evutil_timeradd(&now, tv, &abstime);
        ts.tv_sec = abstime.tv_sec;
        ts.tv_nsec = abstime.tv_usec*1000;
        r = pthread_cond_timedwait(cond, lock, &ts);
        if (r == ETIMEDOUT) return 1;
        else if (r) return -1;
        else return 0;
    } else {
        r = pthread_cond_wait(cond, lock);
        return r ? -1 : 0;
    }
}
```

`evthread_posix_cond_signal` 函数对 `pthread` 的等待通知相关函数进行了封装：

- 当 `tv == 0` 时，相当于调用 `pthread_cond_wait` 函数
- 当 `tv != 0` 时，相当于调用 `pthread_cond_timedwait` 函数
    > 需要注意的是：`evthread_posix_cond_wait` 的时间参数是相对时间，而 `pthread_cond_timedwait` 函数的时间参数是绝对时间，`evthread_posix_cond_wait` 在内部对两种时间进行了转换，要注意在调用 `evthread_posix_cond_wait` 时不要使用绝对时间！

### 定制函数设置相关函数

#### evthread_set_lock_callbacks 和 evthread_set_condition_callbacks

`evthread_set_lock_callbacks` 和 `evthread_set_condition_callbacks` 函数实现极其相似，本文中给出 `evthread_set_lock_callbacks` 的实现代码：

``` cpp
int
evthread_set_lock_callbacks(const struct evthread_lock_callbacks *cbs)
{
    struct evthread_lock_callbacks *target = evthread_get_lock_callbacks();

    if (event_debug_mode_on_) {
        if (event_debug_created_threadable_ctx_) {
            event_errx(1, "evthread initialization must be called BEFORE anything else!");
        }
    }

    if (!cbs) {
        if (target->alloc)
            event_warnx("Trying to disable lock functions after they have been set up will probaby not work.");
        memset(target, 0, sizeof(evthread_lock_fns_));
        return 0;
    }
    if (target->alloc) {
        /* Uh oh; we already had locking callbacks set up.*/
        if (target->lock_api_version == cbs->lock_api_version &&
            target->supported_locktypes == cbs->supported_locktypes &&
            target->alloc == cbs->alloc &&
            target->free == cbs->free &&
            target->lock == cbs->lock &&
            target->unlock == cbs->unlock) {
            /* no change -- allow this. */
            return 0;
        }
        event_warnx("Can't change lock callbacks once they have been initialized.");
        return -1;
    }
    if (cbs->alloc && cbs->free && cbs->lock && cbs->unlock) {
        memcpy(target, cbs, sizeof(evthread_lock_fns_));
        return event_global_setup_locks_(1);
    } else {
        return -1;
    }
}
```

从代码中可以看出 `evthread_set_lock_callbacks` 函数做了以下工作：

1. 获取全局锁结构体地址：`evthread_get_lock_callbacks()`
    > 全局锁结构体在开启、关闭调试锁时所对应的变量不同，具体可参考：{% post_link "源码阅读 libevent - 多线程：调试锁" %}
        * 如果没有开启调试锁，全局锁结构体对应全局变量 evthread_lock_fns_
        * 如果开启调试锁，全局锁结构体对应全局变量 original_lock_fns_
2. 如果传入定制函数结构体指针为空：
    - 如果当前全局锁结构体未设置，则对该结构体进行清 0 操作
    - 如果当前全局锁结构体已设置，则会先输出警告日志后，对该结构体进行清 0 操作
3. 如果传入的结构体指针不为空
    - 如果当前全局锁结构体未设置
        - 结构体中所有函数指针均指向非空，设置全局锁结构体成功
        - 结构体中所有函数指针存在空指针，设置全局锁结构体失败
    - 如果当前全局锁结构体已设置
        - 传入的结构体与当前全局锁结构体完全一致，设置全局锁结构体成功
        - 传入的结构体与当前全局锁结构体不完全一致，设置全局锁结构体失败

#### evthread_set_id_callback

``` cpp
void evthread_set_id_callback(unsigned long (*id_fn)(void)) {
    evthread_id_fn_ = id_fn;
}
```

`evthread_set_id_callback` 函数看上去非常简单，就是让 `_evthread_id_fn` 这个全局变量指向了传入的函数指针参数，在 `evthread_use_pthreads` 函数的实现中，传入参数为 `evthread_posix_get_id`：

``` cpp
static unsigned long evthread_posix_get_id(void) {
    union {
        pthread_t thr;
#if EVENT__SIZEOF_PTHREAD_T > EVENT__SIZEOF_LONG
        ev_uint64_t id;
#else
        unsigned long id;
#endif
    } r;

#if EVENT__SIZEOF_PTHREAD_T < EVENT__SIZEOF_LONG
    memset(&r, 0, sizeof(r));
#endif
    r.thr = pthread_self();
    return (unsigned long)r.id;
}
```

`evthread_posix_get_id` 函数中 `r` 是一个联合体，有两个成员 `thr` 和 `id`，联合体的特点是联合体内部的成员地址是相同的，然后看到 `r.thr = pthread_self()`，这是获取当前线程的 `id` 然后放到 `thr` 中，由于 `r.id` 与 `r.thr` 地址相同，而 `pthread_t` 本身就是 `unsigned long` 类型，因此实际上返回的 `r.id` 就等于获取到的线程 `id`。

由此也可以知道：`evthread_set_id_callback` 函数就是用来指定获取当前线程 `id` 的函数。

## 参考资料

* [1] [Libevent 源码分析 - 多线程、锁、条件变量 (一)_luotuo44 的专栏 - CSDN 博客](https://blog.csdn.net/luotuo44/article/details/38350633)
* [2] <http://www.wangafu.net/~nickm/libevent-book/Ref1_libsetup.html>
