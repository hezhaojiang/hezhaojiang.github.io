---
title: 源码阅读 libevent - 数据结构：尾队列
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: e6338d04
date: 2020-10-25 13:36:35
---
本文分析的数据结构为 `TAILQ` 尾队列。

`libevent` 中 `TAILQ` 尾队列其实就是升级版的 `LIST` 双向链表。

<!--more-->

> 数据结构 `LIST` 双向链表的分析详见：{% post_link "源码阅读 libevent - 数据结构：双向链表" %}

## 尾队列的定义

### 尾队列的定义宏

``` c++
#define TAILQ_HEAD(name, type)                      \
struct name {                                       \
    struct type *tqh_first; /* first element */                         \
    struct type **tqh_last; /* addr of last next element */             \
}

#define TAILQ_ENTRY(type)                           \
struct {                                            \
    struct type *tqe_next;  /* next element */      \
    struct type **tqe_prev; /* address of previous next element */      \
}
```

### 尾队列的节点结构

同双向链表一样，我们通过代码来分析尾队列的节点结构：

``` c++
#include<stdio.h>
#include<stdlib.h>
#include"queue.h"

typedef struct Node {
    int val;
    TAILQ_ENTRY(Node) node;
} Node;
TAILQ_HEAD(Head, Node);

int main(void) {
    struct Head tailQHead;
    TAILQ_INIT(&tailQHead);

    for (int i = 0; i < 3; i++) {
        Node* new_item = malloc(sizeof(Node));
        new_item->val = i;
        TAILQ_INSERT_HEAD(&tailQHead, new_item, node); // 头插法插入新节点
    }

    Node* p;
    printf("[HEAD] %20p [HEAD ADDR] %20p\n\n",tailQHead.tqh_first, &tailQHead);
    TAILQ_FOREACH(p, &tailQHead, node) {
        printf("[NODE] %20d [NODE ADDR] %20p\n", p->val, p);
        printf("[NEXT] %20p [NEXT ADDR] %20p\n", p->node.tqe_next, &p->node.tqe_next);
        printf("[PREV] %20p [PREV ADDR] %20p\n\n", p->node.tqe_prev, &p->node.tqe_prev);
    }
    printf("[TAIL] %20p [HEAD ADDR] %20p\n\n",tailQHead.tqh_last, &tailQHead);
    return 0;
}
```

在该程序中，定义了节点类型为 `Node` 类型，其中包含了一个 `int` 型的 `val` 变量以及 `TAILQ_ENTRY` 所定义的结构体。可以看到，调用 `TAILQ_HEAD` 宏函数时，传入的 `name` 参数 `Head` 最终就成为了 `TAILQ_HEAD` 下结构体类型名。然后用 `struct Head` 来定义一个 `tailQHead` 变量作为尾队列的头节点，其中保存的即是尾队列中的首尾节点信息了。接着就是以头插法形式插入三个节点，然后遍历输出各个节点中关键成员的值与地址，结果如下：

![libevent 尾队列](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201025215558.png)

从代码结果种可得出尾队列结构如下：

``` node
      Head                       Node1                   Node2                   Node3
+--------------+    +----->+--------------+    +-->+--------------+    +-->+--------------+
|   tqh_first  |----+      |    value     |    |   |    value     |    |   |    value     |
|              |<-----+    +--------------+    |   +--------------+    |   +--------------+
+--------------+      |    |   tqe_next   |----+   |   tqe_next   |----+   |   tqe_next   |-----------+
|   tqh_last   |---+  |    |              |<--+    |              |<--+    |              |<---+      |
+--------------+   |  |    +--------------+   |    +--------------+   |    +--------------+    |    -----
                   |  +----|   tqe_prev   |   +----|   tqe_prev   |   +----|   tqe_prev   |    |     ---
                   |       +--------------+        +--------------+        +--------------+    |      -
                   |                                                                           |
                   +---------------------------------------------------------------------------+
```

## 尾队列的访问

### 尾队列的访问宏

``` c++
#define TAILQ_FIRST(head)           ((head)->tqh_first)
#define TAILQ_END(head)             NULL
#define TAILQ_NEXT(elm, field)      ((elm)->field.tqe_next)
#define TAILQ_LAST(head, headname)  (*(((struct headname *)((head)->tqh_last))->tqh_last))
#define TAILQ_PREV(elm, headname, field)    (*(((struct headname *)((elm)->field.tqe_prev))->tqh_last))
#define TAILQ_EMPTY(head)           (TAILQ_FIRST(head) == TAILQ_END(head))

#define TAILQ_FOREACH(var, head, field)             \
    for((var) = TAILQ_FIRST(head); (var) != TAILQ_END(head); (var) = TAILQ_NEXT(var, field))
#define TAILQ_FOREACH_REVERSE(var, head, headname, field)       \
    for((var) = TAILQ_LAST(head, headname); (var) != TAILQ_END(head); (var) = TAILQ_PREV(var, headname, field))
```

### 尾队列的访问宏的作用

从宏定义名即可看出每个宏的作用：

- `TAILQ_FIRST`：获取尾队列的首节点
- `TAILQ_END`：尾队列的末尾，值恒为 `NULL`
- `TAILQ_NEXT`：获取尾队列的后继节点的地址
- `TAILQ_LAST`：获取尾队列的最后一个节点的地址
- `TAILQ_PREV`：获取尾队列的前继节点的地址
- `TAILQ_EMPTY`：判断尾队列是否为空，为空时返回真
- `TAILQ_FOREACH`：遍历尾队列
- `TAILQ_FOREACH_REVERSE`：反向遍历尾队列

> 注意：在使用 `TAILQ_FOREACH` 或 `TAILQ_FOREACH_REVERSE` 遍历尾队列时要避免尾队列的初始化、删除、替换操作！

### 尾队列的访问过程

#### 尾队列的正向遍历

`libevent` 的尾队列正向遍历方式与双向链表遍历方式一致，可参见：{% post_link "源码阅读 libevent - 数据结构：双向链表" %}。

#### 尾队列的尾节点获取

尾队列的尾节点获取方式如下：

``` c++
#define TAILQ_LAST(head, headname)  (*(((struct headname *)((head)->tqh_last))->tqh_last))
```

由尾队列结构结合下图可知，尾队列的尾节点获取可通过以下步骤获取：

1. 通过 `last` 指针获得尾结点的 `next` 指针的地址
2. 通过尾结点的 `next` 指针的地址加上偏移可以直到尾节点的 `prev` 指针地址
3. 通过 `prev` 指针指向的地址获取尾节点的前一个节点的 `next` 指针地址
4. 前一个节点的 `next` 指针指向的地址即为尾节点的地址

``` node
      Head                       Node1                   Node2                   Node3
+--------------+    +----->+--------------+    +-->+--------------+    +>>>+--------------+
|   tqh_first  |----+      |    value     |    |   |    value     |    ③   |    value     |
|              |<-----+    +--------------+    |   +--------------+    |   +--------------+
+--------------+      |    |   tqe_next   |----+   |   tqe_next   |>>>>+   |   tqe_next   |-----------+
|   tqh_last   |>>>+  |    |              |<--+    |              |<<<+    |              |<<<<+      |
+--------------+   |  |    +--------------+   |    +--------------+   ②    +--------------+    |    -----
                   |  +----|   tqe_prev   |   +----|   tqe_prev   |   +<<<<|   tqe_prev   |    |     ---
                   |       +--------------+        +--------------+        +--------------+    |      -
                   |                                                                           |
                   +>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>①>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>+
```

而 `TAILQ_LAST` 在实现时，为了屏蔽头节点和其他节点结构体上的区别，利用了头节点 `first`/`last` 两个指针和其他节点 `next`/`prev` 指针在内存中结构一致的特点，将 `next`/`prev` 结构体强转为了 `first`/`last` 结构体统一了头节点和其他节点在地址查找上的差异。

#### 尾队列的前继节点获取

尾队列的前继节点获取方式如下：

``` c++
#define TAILQ_PREV(elm, headname, field)    (*(((struct headname *)((elm)->field.tqe_prev))->tqh_last))
```

尾队列的前继节点获取和尾队列的尾节点获取原理一致，可参考下图进行理解（获取 `Node3` 的前继节点）：

``` node
      Head                       Node1                   Node2                   Node3
+--------------+    +----->+--------------+    +>>>+--------------+    +-->+--------------+
|   tqh_first  |----+      |    value     |    ③   |    value     |    |   |    value     |
|              |<-----+    +--------------+    |   +--------------+    |   +--------------+
+--------------+      |    |   tqe_next   |>>>>+   |   tqe_next   |----+   |   tqe_next   |-----------+
|   tqh_last   |---+  |    |              |<<<+    |              |<<<+    |              |<---+      |
+--------------+   |  |    +--------------+   ②    +--------------+   ①    +--------------+    |    -----
                   |  +----|   tqe_prev   |   +<<<<|   tqe_prev   |   +<<<<|   tqe_prev   |    |     ---
                   |       +--------------+        +--------------+        +--------------+    |      -
                   |                                                                           |
                   +---------------------------------------------------------------------------+
```

#### 尾队列的反向遍历

反向遍历操作同样有三个关键点：遍历的起始节点，遍历的下一节点的获取，遍历的结束条件。

- 遍历的起始节点：参考尾队列的尾节点获取
- 遍历的下一节点的获取：参考尾队列的前继节点获取
- 遍历的结束条件：`TAILQ_PREV` 获取地址为空，下图为反向遍历到最后一个节点的情况：

    ``` node
          Head                       Node1                   Node2                   Node3
    +--------------+    +----->+--------------+    +-->+--------------+    +-->+--------------+
    |   tqh_first  |----+      |    value     |    |   |    value     |    |   |    value     |
    |              |<<<<<<+    +--------------+    |   +--------------+    |   +--------------+
    +--------------+      |    |   tqe_next   |----+   |   tqe_next   |----+   |   tqe_next   |>>>>>>③>>>>+
    |   tqh_last   |>>>+  ①    |              |<--+    |              |<--+    |              |<<<<+      |
    +--------------+   |  |    +--------------+   |    +--------------+   |    +--------------+    |    -----
                       |  +<<<<|   tqe_prev   |   +----|   tqe_prev   |   +----|   tqe_prev   |    |     ---
                       |       +--------------+        +--------------+        +--------------+    |      -
                       |                                                                           |
                       +>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>②>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>+
    ```

## 尾队列操作

### 初始化

``` c++
#define TAILQ_INIT(head) do {                       \
    (head)->tqh_first = NULL;                       \
    (head)->tqh_last = &(head)->tqh_first;          \
} while (0)
```

### 头插法

``` c++
#define TAILQ_INSERT_HEAD(head, elm, field) do {    \
    if (((elm)->field.tqe_next = (head)->tqh_first) != NULL)    \
        (head)->tqh_first->field.tqe_prev = &(elm)->field.tqe_next;             \
    else (head)->tqh_last = &(elm)->field.tqe_next; \
    (head)->tqh_first = (elm);                      \
    (elm)->field.tqe_prev = &(head)->tqh_first;     \
} while (0)
```

### 尾插法

``` c++
#define TAILQ_INSERT_TAIL(head, elm, field) do {    \
    (elm)->field.tqe_next = NULL;                   \
    (elm)->field.tqe_prev = (head)->tqh_last;       \
    *(head)->tqh_last = (elm);                      \
    (head)->tqh_last = &(elm)->field.tqe_next;      \
} while (0)
```

### 后插法

``` c++
#define TAILQ_INSERT_AFTER(head, listelm, elm, field) do {                      \
    if (((elm)->field.tqe_next = (listelm)->field.tqe_next) != NULL)            \
        (elm)->field.tqe_next->field.tqe_prev = &(elm)->field.tqe_next;         \
    else (head)->tqh_last = &(elm)->field.tqe_next;                             \
    (listelm)->field.tqe_next = (elm);              \
    (elm)->field.tqe_prev = &(listelm)->field.tqe_next;                         \
} while (0)
```

### 前插法

``` c++
#define TAILQ_INSERT_BEFORE(listelm, elm, field) do {                           \
    (elm)->field.tqe_prev = (listelm)->field.tqe_prev;                          \
    (elm)->field.tqe_next = (listelm);              \
    *(listelm)->field.tqe_prev = (elm);             \
    (listelm)->field.tqe_prev = &(elm)->field.tqe_next;                         \
} while (0)
```

### 删除节点

``` c++
#define TAILQ_REMOVE(head, elm, field) do {         \
    if (((elm)->field.tqe_next) != NULL)            \
        (elm)->field.tqe_next->field.tqe_prev = (elm)->field.tqe_prev;          \
    else (head)->tqh_last = (elm)->field.tqe_prev;  \
    *(elm)->field.tqe_prev = (elm)->field.tqe_next; \
} while (0)
```

### 替换节点

``` c++
#define TAILQ_REPLACE(head, elm, elm2, field) do {                              \
    if (((elm2)->field.tqe_next = (elm)->field.tqe_next) != NULL)               \
        (elm2)->field.tqe_next->field.tqe_prev = &(elm2)->field.tqe_next;       \
    else (head)->tqh_last = &(elm2)->field.tqe_next;                            \
    (elm2)->field.tqe_prev = (elm)->field.tqe_prev; \
    *(elm2)->field.tqe_prev = (elm2);               \
} while (0)
```

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
