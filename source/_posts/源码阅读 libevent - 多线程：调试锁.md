---
title: 源码阅读 libevent - 多线程：调试锁
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: a26c1bab
date: 2020-10-25 04:53:10
---
调试锁是 `libevent` 中用户可选的一种模式，它不仅可以调用前面设置的锁函数和条件变量函数，还可以捕获使用锁时的典型错误：重新锁定一个已锁定的非递归锁、解锁一个并未持有的锁。

<!--more-->

## 开启调试锁

开启调试锁的函数 `evthread_enable_lock_debuging` 或 `evthread_enable_lock_debugging，用户可直接调用，其定义如下：`

> `evthread_enable_lock_debuging` 函数为 `evthread_enable_lock_debugging` 函数的拼写错误版本。因兼容性原因进行保留。

``` c++
GLOBAL int evthread_lock_debugging_enabled_ = 0;

void evthread_enable_lock_debuging(void) {
    evthread_enable_lock_debugging();
}

void evthread_enable_lock_debugging(void) {
    struct evthread_lock_callbacks cbs = {
        EVTHREAD_LOCK_API_VERSION,
        EVTHREAD_LOCKTYPE_RECURSIVE,
        debug_lock_alloc,
        debug_lock_free,
        debug_lock_lock,
        debug_lock_unlock
    };
    if (evthread_lock_debugging_enabled_) return;
    memcpy(&original_lock_fns_, &evthread_lock_fns_, sizeof(struct evthread_lock_callbacks));
    memcpy(&evthread_lock_fns_, &cbs, sizeof(struct evthread_lock_callbacks));
    memcpy(&original_cond_fns_, &evthread_cond_fns_, sizeof(struct evthread_condition_callbacks));
    evthread_cond_fns_.wait_condition = debug_cond_wait;
    evthread_lock_debugging_enabled_ = 1;
    event_global_setup_locks_(0);
}
```

在该函数中做了如下工作：

1. 定义了一个锁函数结构体 `cbs`，其中已经设定好了一系列的调试锁相关的锁函数。
2. 判断 `_evthread_lock_debugging_enabled` 是否为真
    - 如果 `_evthread_lock_debugging_enabled` 是否为真，说明已经开启了调试锁，该函数直接返回
    - 如果 `_evthread_lock_debugging_enabled` 是否为假，则继续以下工作
3. 备份全局锁和条件变量结构体：
    - 将 `evthread_lock_fns_` 拷贝至 `original_lock_fns_`
    - 将 `evthread_cond_fns_` 拷贝至 `original_cond_fns_`
4. 重新设置全局锁和条件变量结构体：
    - 将 `cbs` 拷贝至 `evthread_lock_fns_`
    - 将 `evthread_cond_fns_.wait_condition` 赋值为 `debug_cond_wait`

## 调试锁结构

``` c++
struct debug_lock {
    unsigned signature;
    unsigned locktype;
    unsigned long held_by;
    /* XXXX if we ever use read-write locks, we will need a separate lock to protect count. */
    int count;
    void *lock;
};
```

调试锁结构体中有 5 个成员：

- `signature`：对调试锁结构进行签名，符合签名的才被认为是合法的调试锁结构
- `locktype`：来保存用户申请的锁类型，因为调试锁中所有的锁都增加了递归属性，所以需要对原始锁类型进行保存
- `count`：对加锁次数进行计数，如果 `count` 大于 `1` 说明多次加锁，如果 `count` 小于 `0` 说明多次解锁
- `held_by`：记录持有锁的线程
- `lock`：锁变量

## 调试锁函数

### debug_lock_alloc

``` c++
static void * debug_lock_alloc(unsigned locktype) {
    struct debug_lock *result = mm_malloc(sizeof(struct debug_lock));
    if (!result) return NULL;
    if (original_lock_fns_.alloc) {
        if (!(result->lock = original_lock_fns_.alloc(locktype|EVTHREAD_LOCKTYPE_RECURSIVE))) {
            mm_free(result);
            return NULL;
        }
    } else result->lock = NULL;
    result->signature = DEBUG_LOCK_SIG;
    result->locktype = locktype;
    result->count = 0;
    result->held_by = 0;
    return result;
}
```

`debug_lock_alloc` 函数用来初始化一个调试锁，其主要工作分为三部分：

1. 申请调试锁结构体内存空间
2. 调用用户定制的锁初始化函数初始化一个锁（注意此时初始化的锁增加了递归属性）
3. 对调试锁结构体中的每个参数进行初始化

### debug_lock_free

``` c++
static void debug_lock_free(void *lock_, unsigned locktype) {
    struct debug_lock *lock = lock_;
    EVUTIL_ASSERT(lock->count == 0);
    EVUTIL_ASSERT(locktype == lock->locktype);
    EVUTIL_ASSERT(DEBUG_LOCK_SIG == lock->signature);
    if (original_lock_fns_.free) {
        original_lock_fns_.free(lock->lock, lock->locktype|EVTHREAD_LOCKTYPE_RECURSIVE);
    }
    lock->lock = NULL;
    lock->count = -100;
    lock->signature = 0x12300fda;
    mm_free(lock);
}
```

`debug_lock_alloc` 函数用来销毁一个调试锁，其主要工作分为三部分：

1. 调试锁参数的检查
    > 调试锁参数的检查过程中调用了 `EVUTIL_ASSERT` 宏，该宏是 `libevent` 对 `assert` 函数的一个封装，功能类似。
2. 调用用户定制的锁销毁函数销毁一个锁（注意此时销毁的锁增加了递归属性）
3. 释放调试锁结构体内存空间

### debug_lock_lock

``` c++
static int debug_lock_lock(unsigned mode, void *lock_) {
    struct debug_lock *lock = lock_;
    int res = 0;
    if (lock->locktype & EVTHREAD_LOCKTYPE_READWRITE) EVUTIL_ASSERT(mode & (EVTHREAD_READ|EVTHREAD_WRITE));
    else EVUTIL_ASSERT((mode & (EVTHREAD_READ|EVTHREAD_WRITE)) == 0);
    if (original_lock_fns_.lock) res = original_lock_fns_.lock(mode, lock->lock);
    if (!res) evthread_debug_lock_mark_locked(mode, lock);
    return res;
}
```

`debug_lock_lock` 函数用来对一个调试锁执行加锁操作，其主要工作分为三部分：

1. 参数的检查：
    - 如果锁模式是读写锁，则 `mode` 类型必须携带 `EVTHREAD_READ` 或 `EVTHREAD_WRITE` 标志位
    - 如果锁模式是非读写锁，则 `mode` 类型不能携带 `EVTHREAD_READ` 和 `EVTHREAD_WRITE` 标志位
2. 调用用户定制的加锁函数进行加锁
3. 如果加锁成功则调用 `evthread_debug_lock_mark_locked` 函数检测加锁错误并对调试锁结构体进行修改

#### evthread_debug_lock_mark_locked

``` c++
static void evthread_debug_lock_mark_locked(unsigned mode, struct debug_lock *lock) {
    EVUTIL_ASSERT(DEBUG_LOCK_SIG == lock->signature);
    ++lock->count;
    if (!(lock->locktype & EVTHREAD_LOCKTYPE_RECURSIVE)) EVUTIL_ASSERT(lock->count == 1);
    if (evthread_id_fn_) {
        unsigned long me;
        me = evthread_id_fn_();
        if (lock->count > 1) EVUTIL_ASSERT(lock->held_by == me);
        lock->held_by = me;
    }
}
```

`evthread_debug_lock_mark_locked` 函数有两个作用：检测加锁错误并对调试锁结构体进行修改。

检测的加锁错误有以下三种：

1. 初始化检测 / 锁变量检测：调试锁签名不正确
2. 重复加锁检测：对于非递归锁，加锁后 `count` 值不为 `1`
3. 不同线程申请同一递归锁检测：`count` 值大于 `1` 的情况下，`held_by` 的线程 `id` 和当前线程 `id` 不一致

修改调试锁结构体：

1. `count` 值自增
2. `held_by` 值设置为当前线程 `id`

### debug_lock_unlock

``` c++
static int debug_lock_unlock(unsigned mode, void *lock_) {
    struct debug_lock *lock = lock_;
    int res = 0;
    evthread_debug_lock_mark_unlocked(mode, lock);
    if (original_lock_fns_.unlock) res = original_lock_fns_.unlock(mode, lock->lock);
    return res;
}
```

调试锁的解锁函数也是会调用用户定制的解锁函数，需要注意调试锁的解锁与加锁流程区别：调试锁的解锁操作会在调用用户定制的解锁函数之前，先调用 `evthread_debug_lock_mark_unlocked` 函数进行解锁检测，而调试锁的加锁操作是在调用用户定制的加锁函数后调用加锁检测函数。

#### evthread_debug_lock_mark_unlocked

``` c++
static void evthread_debug_lock_mark_unlocked(unsigned mode, struct debug_lock *lock) {
    EVUTIL_ASSERT(DEBUG_LOCK_SIG == lock->signature);
    if (lock->locktype & EVTHREAD_LOCKTYPE_READWRITE) EVUTIL_ASSERT(mode & (EVTHREAD_READ|EVTHREAD_WRITE));
    else EVUTIL_ASSERT((mode & (EVTHREAD_READ|EVTHREAD_WRITE)) == 0);
    if (evthread_id_fn_) {
        unsigned long me;
        me = evthread_id_fn_();
        EVUTIL_ASSERT(lock->held_by == me);
        if (lock->count == 1) lock->held_by = 0;
    }
    --lock->count;
    EVUTIL_ASSERT(lock->count >= 0);
}
```

`evthread_debug_lock_mark_unlocked` 函数作用与 `evthread_debug_lock_mark_locked` 函数类似：检测解锁错误并对调试锁结构体进行修改。

检测的解锁错误有以下四种：

1. 初始化检测 / 锁变量检测：调试锁签名不正确
2. 解锁模式的检测：加锁时对加锁模式的检测是在 `debug_lock_lock` 函数中
3. 解锁线程的检测：加解锁必须为同一个线程
4. 加锁次数检测：未加锁的锁禁止解锁

修改调试锁结构体：

1. `count` 值自减
2. 如果解锁后该锁完全解锁，则将 `held_by` 值设置 `0`

### debug_cond_wait

``` c++
static int debug_cond_wait(void *cond_, void *lock_, const struct timeval *tv) {
    int r;
    struct debug_lock *lock = lock_;
    EVUTIL_ASSERT(lock);
    EVUTIL_ASSERT(DEBUG_LOCK_SIG == lock->signature);
    EVLOCK_ASSERT_LOCKED(lock_);
    evthread_debug_lock_mark_unlocked(0, lock);
    r = original_cond_fns_.wait_condition(cond_, lock->lock, tv);
    evthread_debug_lock_mark_locked(0, lock);
    return r;
}
```

条件变量函数中，`alloc`、`free`、`wait`、`signal` 四个函数，其中 `alloc`、`free`、`signal` 都仅仅涉及到条件变量的操作，而 `wait` 函数却不同，`wait` 函数需要将已加锁的锁先进行解锁然后阻塞，直到等到唤醒消息时再重新持有锁，所以在调试锁时，有必要对 `wait` 函数进行修改。

`debug_cond_wait` 函数的作用比较简单：

1. 对基本参数进行检测
2. 调用 `evthread_debug_lock_mark_unlocked` 检测解锁错误并对调试锁结构体进行修改
3. 调用用户定制的条件变量 `wait` 函数等待条件唤醒
4. 调用 `evthread_debug_lock_mark_locked` 检测加锁错误并对调试锁结构体进行修改

## 参考资料

* [1] [libevent 源码学习（4）：线程锁、条件变量（二）（调试锁）_HerofH_的博客 - CSDN 博客](https://blog.csdn.net/qq_28114615/article/details/90451936)
* [2] [Libevent 源码分析 - 多线程、锁、条件变量 (二)_luotuo44 的专栏 - CSDN 博客](https://blog.csdn.net/luotuo44/article/details/38360525)
