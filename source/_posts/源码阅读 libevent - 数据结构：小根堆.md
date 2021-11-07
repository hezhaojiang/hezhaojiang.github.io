---
title: 源码阅读 libevent - 数据结构：小根堆
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: 3a941cca
date: 2020-11-02 14:00:47
---
在 `libevent` 中，使用 `min_heap` 这一数据结构来管理各个 `event` 的超时，也就是小根堆，整个堆是根据各个 `event` 的超时时间来构成的，因此堆顶肯定就对应超时时间最小的 `event`，这样就可以按照超时顺序进行处理了。

<!--more-->

## 堆的数组实现

堆 (Heap) 是一种数据结构，其由两个特点：

- 堆中某个节点的值总是不大于或不小于其父节点的值
    - 大根堆（大顶堆）：堆中节点的值总是不大于其父节点的值
    - 小根堆（小顶堆）：堆中节点的值总是不小于其父节点的值
- 堆总是一棵完全二叉树

而完全二叉树由于其特性，通常由数组来实现：

``` c++
/* 已知一个节点所在数组下标，获取其左子节点、右子节点、父节点的数组下标 */
#define GetLeftChild(index)     ((index) * 2 + 1)
#define GetRightChild(index)    ((index) * 2 + 2)
#define GetParent(index)        (((index) - 1) / 2)
/* 已知一个节点所在数组下标和二叉树节点个数，获取其左子节点、右子节点、父节点是否存在 */
#define hasLeftChild(index, size)   (GetLeftChild(index) < (size))
#define hasRightChild(index, size)  (hasRightChild(index) < (size))
#define hasParent(index, size)      ((index) > 0)
```

## min_heap 的定义

``` c++
typedef struct min_heap
{
    struct event** p;
    unsigned n, a;
} min_heap_t;
```

各成员变量意义如下：

1. `p`：二级指针，指向小根堆的首地址，小根堆为一个数组，数组元素结构为 `struct event*`
2. `n`：数组中 `struct event*` 有效元素个数
3. `a`：`struct event*` 小根堆的容量，即数组的容量

## min_heap 的优先级对比

要实现堆，必须要定义队中节点优先级的比较方式，`libevent` 中 min_heap 中保存的是每个 `event` 的超时时间，因此需要根据超时顺序进行排列，其中 `min_heap_elem_greater` 就是用来比较两个 `event` 的超时时间的函数。

`min_heap_elem_greater` 函数传入两个 `event` 参数，用来判断第一个参数 `event` 的超时结构体是否大于第二参数的超时结构体，如果大于则返回 `1`，否则返回 `0`。比较两个超时结构体先比较秒数，再比较微妙数，函数中调用了宏函数，如下所示：

``` c++
#define evutil_timercmp(tvp, uvp, cmp)                  \
    (((tvp)->tv_sec == (uvp)->tv_sec) ? ((tvp)->tv_usec cmp (uvp)->tv_usec) : ((tvp)->tv_sec cmp (uvp)->tv_sec))

#define min_heap_elem_greater(a, b) (evutil_timercmp(&(a)->ev_timeout, &(b)->ev_timeout, >))
```

``` c++
struct event {
    ......
    union {
        ......
        int min_heap_idx;
    } ev_timeout_pos;
```

## min_heap 的初始化

虽然说 `C` 语言中没有构造函数和析构函数，但是 `min_heap` 也将这种思想进行了体现在了 `min_heap_ctor` 函数和 `min_heap_dtor` 函数上，从函数名上看就是 `constructor` 和 `destructor` 的简写，各自定义如下：

``` c++
void min_heap_ctor_(min_heap_t* s) { s->p = 0; s->n = 0; s->a = 0; }
void min_heap_dtor_(min_heap_t* s) { if (s->p) mm_free(s->p); }
```

`min_heap_elem_init` 函数用来初始化小根堆中的 `event`，将 `event` 的堆索引初始化为 `-1`。其定义如下：

``` c++
void min_heap_elem_init_(struct event* e) { e->ev_timeout_pos.min_heap_idx = -1; }
```

## min_heap 的重要操作

一个最小堆数据结构体，其最重要的操作就是实现 `push`/`pop`/`peek`：

- `push`：将一个新的节点放入堆中
- `pop`：将堆中最小节点弹出
- `peek`：获取堆中最小节点

### push

首先来看下向堆中插入节点的一般实现步骤：

1. 在堆的最后新建一个节点
2. 将数值赋予新节点
3. 将其与父节点比较
4. 如果新节点优先级比父节点高，则调换父子节点位置
5. 重复步骤 3 和步骤 4 直至堆特性被满足

接下来我们再分析下 `libevent` 向堆中插入一个新节点的过程，其由 `min_heap_push_` 函数实现，其定义如下：

``` c++
int min_heap_push_(min_heap_t* s, struct event* e) {
    if (s->n == UINT32_MAX || min_heap_reserve_(s, s->n + 1)) return -1;
    min_heap_shift_up_(s, s->n++, e);
    return 0;
}
```

其中涉及到调用两个函数 `min_heap_reserve_` 和 `min_heap_shift_up_`，函数功能如下：

1. `min_heap_reserve_`：为新插入元素分配足够大小的堆内存，为上述五个步骤开辟空间
2. `min_heap_shift_up_`：堆元素上浮操作，其中包含了上述五个步骤中的步骤 3 到步骤 5

> 步骤 1 和步骤 2 由用户开辟空间并进行赋值

#### 分配堆空间

`min_heap` 是在插入新的 `event` 时，如果空间不足是可以自动扩容的，该函数需要传入 `n` 表明需要让堆装下 `n` 个元素。由函数 `min_heap_reserve_` 实现如下：

``` c++
int min_heap_reserve_(min_heap_t* s, unsigned n) {
    if (s->a < n) {
        struct event** p;
        unsigned a = s->a ? s->a * 2 : 8;
        if (a < n) a = n;
#if (SIZE_MAX == UINT32_MAX)
        if (a> SIZE_MAX / sizeof *p) return -1;
#endif
        if (!(p = (struct event**)mm_realloc(s->p, a * sizeof *p))) return -1;
        s->p = p;
        s->a = a;
    }
    return 0;
}
```

前面说过，`min_heap` 的成员变量 `a` 描述的是堆最大所能容纳的元素数目，也就是堆的容量。如果传入的 `n` 本身小于 `a`，说明当前堆完全可以装下 `n` 个元素，因此无需再扩容了。

如果传入的 `n` 不小于 `a`，说明此时的堆刚刚能装下或者装不下 `n` 个元素，此时就需要对堆进行扩容。`min_heap` 这里分了两种情况：如果堆本身为空，那么就直接为堆分配 `8` 个元素的空间；如果堆本身不为空，那么就先将堆原本的空间加倍，作为堆的新容量，如果堆非空时加倍之后或者堆空时分配 `8` 个元素空间还放不下 `n` 个元素，那么就直接把 `n` 作为堆的新容量。这样做的好处是不用每次插入一个新的 `event` 都去重新分配空间。

此外，如果 `min_heap` 需要分配更大的空间，这里使用的是 `realloc` 函数，会先调用 `malloc` 函数进行指定大小空间的分配，再把原来的内存数据复制到新空间中。

#### 堆元素的上浮

在小根堆中，当堆中元素需要进行调整时，就会对相应的元素进行上浮或者下沉，之所以要这样做，是因为堆中元素调整后不一定还满足小根堆的性质，因此就要重新进行调整，让堆重新满足原来的特性，其对应着上述五个步骤中的步骤 3 到步骤 5。

`min_heap` 的堆元素上浮是通过 `min_heap_shift_up_` 函数实现的，该函数定义如下：

``` c++
void min_heap_shift_up_(min_heap_t* s, unsigned hole_index, struct event* e) {
    unsigned parent = (hole_index - 1) / 2;
    while (hole_index && min_heap_elem_greater(s->p[parent], e)) {
        (s->p[hole_index] = s->p[parent])->ev_timeout_pos.min_heap_idx = hole_index;
        hole_index = parent;
        parent = (hole_index - 1) / 2;
    }
    (s->p[hole_index] = e)->ev_timeout_pos.min_heap_idx = hole_index;
}
```

### pop

首先来看下向堆中弹出节点的一般实现步骤：

1. 移除根节点
2. 将最后一个节点移到根节点处
3. 将子节点优先级较高者与父节点比较
4. 如果父节点优先级比子节点低，则调换父子节点位置
5. 重复步骤 3 和步骤 4 直至堆特性被满足

接下来我们再分析下 `libevent` 向堆中弹出一个新节点的过程，，其由 `min_heap_pop_` 函数实现，其定义如下：

``` c++
struct event* min_heap_pop_(min_heap_t* s) {
    if (s->n) {
        struct event* e = *s->p;
        min_heap_shift_down_(s, 0u, s->p[--s->n]);
        e->ev_timeout_pos.min_heap_idx = -1;
        return e;
    }
    return 0;
}
```

#### 堆元素的下沉

与堆元素的上浮相似，由 `min_heap_shift_down_` 函数实现，其定义如下：

``` c++
void min_heap_shift_down_(min_heap_t* s, unsigned hole_index, struct event* e) {
    unsigned min_child = 2 * (hole_index + 1);
    while (min_child <= s->n) {
        min_child -= min_child == s->n || min_heap_elem_greater(s->p[min_child], s->p[min_child - 1]);
        if (!(min_heap_elem_greater(e, s->p[min_child]))) break;
        (s->p[hole_index] = s->p[min_child])->ev_timeout_pos.min_heap_idx = hole_index;
        hole_index = min_child;
        min_child = 2 * (hole_index + 1);
    }
    (s->p[hole_index] = e)->ev_timeout_pos.min_heap_idx = hole_index;
}
```

### peek

根据堆的特性，`peek` 操作只需要返回非空数组的首个元素即可：

``` c++
struct event* min_heap_top(min_heap_t* s) { return s->n ? *s->p : 0; }
```

## min_heap 的其他操作

`min_heap_elt_is_top` 函数用于判断 `event` 是否在堆顶。显然，如果 `event` 的堆索引为 `0`，那么这个 `event` 就在堆顶了。其定义如下：

``` c++
int min_heap_elt_is_top_(const struct event *e) {
    return e->ev_timeout_pos.min_heap_idx == 0;
}
```

### 判断堆是否为空及堆大小

前面说了 `min_heap` 中的成员变量 `n` 描述堆中实际存在的元素数目，因此直接判断 `n` 是否为 `0` 即可：

``` c++
int min_heap_empty(min_heap_t* s) { return 0u == s->n; }   // 堆是否为空
unsigned min_heap_size(min_heap_t* s) { return s->n; }   // 堆大小
```

### 堆删除元素

需要注意的一点是，由于堆末尾的元素对于整个堆来说，删除它对于堆是没有任何影响的，因此，如果要对堆中的任意一个元素进行删除，就可以将需要删除的元素先和堆尾元素互换，然后不考虑需要删除的元素，对互换后的堆进行调整，最终得到的堆就是删除了该元素的堆了。由 `min_heap_erase_` 实现，由于其定义如下：

``` c++
int min_heap_erase_(min_heap_t* s, struct event* e) {
    if (-1 != e->ev_timeout_pos.min_heap_idx) {
        struct event *last = s->p[--s->n];
        unsigned parent = (e->ev_timeout_pos.min_heap_idx - 1) / 2;
        if (e->ev_timeout_pos.min_heap_idx > 0 && min_heap_elem_greater(s->p[parent], last))
            min_heap_shift_up_unconditional_(s, e->ev_timeout_pos.min_heap_idx, last);
        else
            min_heap_shift_down_(s, e->ev_timeout_pos.min_heap_idx, last);
        e->ev_timeout_pos.min_heap_idx = -1;
        return 0;
    }
    return -1;
}

void min_heap_shift_up_unconditional_(min_heap_t* s, unsigned hole_index, struct event* e) {
    unsigned parent = (hole_index - 1) / 2;
    do {
        (s->p[hole_index] = s->p[parent])->ev_timeout_pos.min_heap_idx = hole_index;
        hole_index = parent;
        parent = (hole_index - 1) / 2;
    } while (hole_index && min_heap_elem_greater(s->p[parent], e));
    (s->p[hole_index] = e)->ev_timeout_pos.min_heap_idx = hole_index;
}
```

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
* [2] [libevent 源码学习（10）：min_heap 数据结构解析_HerofH_的博客 - CSDN 博客](https://blog.csdn.net/qq_28114615/article/details/95342338)
