---
title: 源码阅读 libevent - 内存管理
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 58deb66e
date: 2020-10-21 13:37:19
---
`libevent` 的内存管理模块的头文件为 `mm-internal.h`，而相关函数的定义在文件 `event_mm_malloc_` 中。

从头文件名就可以看出，同日志模块一样，`libevent` 的内存管理相关函数也只提供 `libevent` 内部使用，并不提供外部接口。

同时，也同日志模块一样，`libevent` 的内存管理模块也向用户提供了设置回调函数的接口，以方便用户自行实现内存管理函数。

<!--more-->

## 内存管理函数

### 函数声明

从 `mm-internal.h` 文件中可以得到内存管理模块函数的声明：

``` c++
EVENT2_EXPORT_SYMBOL void *event_mm_malloc_(size_t sz);
EVENT2_EXPORT_SYMBOL void *event_mm_calloc_(size_t count, size_t size);
EVENT2_EXPORT_SYMBOL char *event_mm_strdup_(const char *str);
EVENT2_EXPORT_SYMBOL void *event_mm_realloc_(void *p, size_t sz);
EVENT2_EXPORT_SYMBOL void event_mm_free_(void *p);

#define mm_malloc(sz) event_mm_malloc_(sz)
#define mm_calloc(count, size) event_mm_calloc_((count), (size))
#define mm_strdup(s) event_mm_strdup_(s)
#define mm_realloc(p, sz) event_mm_realloc_((p), (sz))
#define mm_free(p) event_mm_free_(p)
```

注意，以上声明是在 `libevent` 编译时没有使用 `--disable-malloc-replacement` 参数的声明，如果 `libevent` 编译时使用了 `--disable-malloc-replacement` 参数，即：

``` bash
# disable support for replacing the memory mgt functions
./configure --disable-malloc-replacement
```

则 `libevent` 的内存管理函数使用系统的内存管理函数，即：

``` c++
#define mm_malloc(sz) malloc(sz)
#define mm_calloc(n, sz) calloc((n), (sz))
#define mm_strdup(s) strdup(s)
#define mm_realloc(p, sz) realloc((p), (sz))
#define mm_free(p) free(p)
```

### 函数定义

#### mm_malloc & mm_free & mm_realloc

``` c++
#define mm_malloc(sz) event_mm_malloc_(sz)
#define mm_free(p) event_mm_free_(p)
#define mm_realloc(p, sz) event_mm_realloc_((p), (sz))

void *event_mm_malloc_(size_t sz) {
    if (sz == 0) return NULL;
    if (mm_malloc_fn_) return mm_malloc_fn_(sz);
    else return malloc(sz);
}

void event_mm_free_(void *ptr) {
    if (mm_free_fn_) mm_free_fn_(ptr);
    else free(ptr);
}

void *event_mm_realloc_(void *ptr, size_t sz)
{
    if (mm_realloc_fn_) return mm_realloc_fn_(ptr, sz);
    else return realloc(ptr, sz);
}
```

`mm_malloc` & `mm_free` & `mm_realloc` 的实现最为简单，实现方式是有回调函数调用回调函数，没有回调函数直接调用系统函数。

### mm_strdup

``` c++
char *event_mm_strdup_(const char *str) {
    if (!str) {
        errno = EINVAL;
        return NULL;
    }

    if (mm_malloc_fn_) {
        size_t ln = strlen(str);
        void *p = NULL;
        if (ln == EV_SIZE_MAX) goto error;
        p = mm_malloc_fn_(ln+1);
        if (p) return memcpy(p, str, ln+1);
    }
    else return strdup(str);
error:
    errno = ENOMEM;
    return NULL;
}
```

`strdup` 的功能：`strdup` 是指新开辟一段空间，并将传入的字符串复制到新开辟的空间中，并返回这段空间的指针。

该函数的实现在 `_mm_malloc_fn` 为空的情形下调用系统函数 `strdup()` 来实现 `strdup` 功能。

而在 `_mm_malloc_fn` 不为空时的实现的方法是：先获取字符串长度，再加上一个终止符作为总长度，开辟该长度的空间，并取得指向该空间的指针，然后将字符串复制到这段空间。

### mm_calloc

``` c++
void *event_mm_calloc_(size_t count, size_t size) {
    if (count == 0 || size == 0) return NULL;

    if (mm_malloc_fn_) {
        size_t sz = count * size;
        void *p = NULL;
        if (count> EV_SIZE_MAX / size) goto error;
        p = mm_malloc_fn_(sz);
        if (p) return memset(p, 0, sz);
    }
    else {
        void *p = calloc(count, size);
        return p;
    }
error:
    errno = ENOMEM;
    return NULL;
}
```

`calloc` 功能：`calloc` 和 `malloc` 函数相似，都是开辟一段连续的空间，不同点在于 `calloc` 函数需要传入 `count` 和 `size` 两个参数，用来开辟 `count` 个大小为 `size` 的连续空间 (总大小为 `count*size`)，并且为这些空间全部赋值为 `0`。然后返回指向这段空间的泛型指针。

该函数的实现在 `_mm_malloc_fn` 为空的情形下调用系统函数 `calloc()` 来实现 `calloc` 功能。

而在 `_mm_malloc_fn` 不为空时的实现的方法是：先计算开辟空间总大小 `sz`，然后调用 `_mm_malloc_fn` 所指向的函数，而调用的函数也应当实现 `malloc` 的效果，开辟一段空间，然后再用 `memset` 对这段空间初始化为 `0`，并返回指向这段空间的泛型指针。

## 定制内存管理函数接口

`libevent` 库提供了修改 `mm_malloc` & `mm_free` & `mm_realloc` 内存管理方式的函数 `event_set_mem_functions`：

``` c++
void event_set_mem_functions(void *(*malloc_fn)(size_t sz),
            void *(*realloc_fn)(void *ptr, size_t sz),
            void (*free_fn)(void *ptr)) {
    mm_malloc_fn_ = malloc_fn;
    mm_realloc_fn_ = realloc_fn;
    mm_free_fn_ = free_fn;
}
```

`event_set_mem_functions` 的声明位于文件 `event.h`，故可以被用户调用。

通过 `event_set_mem_functions` 函数，设置 `mm_malloc_fn_`，`mm_realloc_fn_`，`mm_free_fn_`，后，可以由用户自行对内存使用进行管理，比如使用内存池等。

在自行定制内存管理函数时应当注意以下问题：

- 替换内存管理函数影响 `libevent` 随后的所有分配、调整大小和释放内存操作。所以必须保证在调用任何其他 `libevent` `函数之前进行定制。否则，libevent` 可能用定制的 `free` 函数释放 `C` 语言库的 `malloc` 函数分配的内存
- `malloc` 和 `realloc` 函数返回的内存块应该具有和 `C` 库返回的内存块一样的地址对齐
- `realloc` 函数应该正确处理 `realloc(NULL, sz)`（也就是当作 `malloc(sz)` 处理）
- `realloc` 函数应该正确处理 `realloc(ptr, 0)`（也就是当作 `free(ptr)` 处理）
- 如果在多个线程中使用 `libevent`，替代的内存管理函数需要是线程安全的
- 如果要释放由 `libevent` 函数分配的内存，并且已经定制了 `malloc` 和 `realloc` 函数，那么就应该使用定制的 `free` 函数释放。否则将会 `C` 语言标准库的 `free` 函数释放定制内存分配函数分配的内存，这将发生错误。所以三者要么全部不定制，要么全部定制。

## 参考资料

* [1] [Libevent 源码分析 - 内存分配_luotuo44 的专栏 - CSDN 博客](https://blog.csdn.net/luotuo44/article/details/38334979)
* [2] <http://www.wangafu.net/~nickm/libevent-book/Ref1_libsetup.html>
