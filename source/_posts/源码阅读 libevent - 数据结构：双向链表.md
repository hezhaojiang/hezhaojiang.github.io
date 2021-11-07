---
title: 源码阅读 libevent - 数据结构：双向链表
categories:
  - 源码阅读
tags:
  - libevent
abbrlink: ee7292d
date: 2020-10-25 08:27:29
---
`libevent` 源码中有一个 `queue.h` 文件，该文件里面定义了 5 种数据结构：

- `SLIST` - 单向链表
- `LIST` - 双向链表
- `SIMPLEQ` - 简单队列
- `TAILQ` - 尾队列
- `CIRCLEQ` - 环形队列

其中 `SLIST`/`SIMPLEQ`/`CIRCLEQ` 在 `libevent` 源码中没有被使用，所以本系列文章针对 `LIST`/`TAILQ` 进行一下分析。

本文分析的数据结构为 `LIST` 双向链表。

<!--more-->

> 数据结构 `TAILQ` 尾队列的分析详见：{% post_link "源码阅读 libevent - 数据结构：尾队列" %}

libevent 中，5 种数据结构的各类操作均通过宏定义实现，且所有宏定义按功能分成了三部分：

1. 数据结构定义宏
2. 数据结构访问宏
3. 数据结构操作宏

## 双向链表的定义

### 双向链表的定义宏

``` cpp
#define LIST_HEAD(name, type)                       \
struct name {                                       \
    struct type *lh_first;  /* first element */     \
}

#define LIST_ENTRY(type)                            \
struct {                                            \
    struct type *le_next;   /* next element */      \
    struct type **le_prev;  /* address of previous next element */      \
}
```

### 双向链表的定义宏的作用

先来猜测一下这两个宏定义的作用。在宏定义 `LIST_HEAD` 下的结构体中，只包含了一个结构体成员 `lh_first`，从成员名就能推测，`lh_first` 应当是和链表第一个元素有关。再来看宏定义 `LIST_ENTRY`，其中也包含了两个结构体成员 `le_next` 和 `le_prev`，从变量名就会发现，二者应当与前后元素相关。因此，就可以推断，宏定义中所需输入的参数 `type` 实际上就是节点类型，这个节点类型应该包含但不限于 `LIST_ENTRY` 所定义的结构，而 `LIST_HEAD` 则是对于整个双向链表而言的，用于找到首节点元素，因此 `LIST_HEAD` 中的 `type` 也应该是与 `LIST_ENTRY` 中相同的节点类型。

那么这里为什么 `LIST_HEAD` 还需要一个参数 `name` 呢？前面说了，`LIST_ENTRY` 应当包含在节点类型的定义中，节点类型一旦定义好了并定义了一个节点，那么自然而然 `le_next` 和 `le_prev` 就都包含在该节点中了，此时 `LIST_ENTRY` 结构体作为一个匿名结构体即可，因此无需指定 `name` 来定义 `LIST_ENTRY` 结构体的名称；而对于 `LIST_HEAD` 来说，它是独立的数据类型，用来描述了双向链表的首尾节点，需要用 `LIST_HEAD` 来定义一个具体的结构体来存放首尾节点指针，因此这里必须指明结构体名 `name`。

根据前面的推测，现在来正式分析一下 `LIST_HEAD` 与 `LIST_ENTRY` 中各成员的含义。

对于 `LIST_HEAD` 宏定义，其中的 `lh_first` 为一级指针，也就是说，`lh_first` 指向一个 `struct type` 类型的节点。`TAILQ_ENTRY` 中的 le_next 也是类似，`le_prev` 则是指向一个指向一个 `struct type` 类型的节点的指针。

我们可以写一个测试程序来看看该双向链表的结构：

``` cpp
#include<stdio.h>
#include<stdlib.h>
#include"queue.h"

typedef struct Node {
    int val;
    LIST_ENTRY(Node) node;
} Node;
LIST_HEAD(Head, Node);

int main(void) {
    struct Head listHead;
    LIST_INIT(&listHead);

    for (int i = 0; i < 3; i++) {
        Node* new_item = malloc(sizeof(Node));
        new_item->val = i;
        LIST_INSERT_HEAD(&listHead, new_item, node); // 头插法插入新节点
    }

    Node* p;
    printf("[HEAD] %20p [HEAD ADDR] %20p\n\n",listHead.lh_first, &listHead);
    LIST_FOREACH(p, &listHead, node) {
        printf("[NODE] %20d [NODE ADDR] %20p\n", p->val, p);
        printf("[NEXT] %20p [NEXT ADDR] %20p\n", p->node.le_next, &p->node.le_next);
        printf("[PREV] %20p [PREV ADDR] %20p\n\n", p->node.le_prev, &p->node.le_prev);
    }
    return 0;
}
```

在该程序中，定义了节点类型为 `Node` 类型，其中包含了一个 `int` 型的 `val` 变量以及 `LIST_ENTRY` 所定义的结构体。可以看到，调用 `LIST_HEAD` 宏函数时，传入的 `name` 参数 `Head` 最终就成为了 `LIST_HEAD` 下结构体类型名。然后用 `struct Head` 来定义一个 `listHead` 变量，其中保存的即是双向链表中的首节点信息了。接着就是以头插法形式插入三个节点，然后遍历输出各个节点中关键成员的值与地址，结果如下：

![libevent 双向链表](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201025200734.jpg)

从代码结果种可得出链表结构如下：

``` node
      Head                   Node1                   Node2                   Node3
+--------------+    +-->+--------------+    +-->+--------------+    +-->+--------------+
|   lh_first   |----+   |    value     |    |   |    value     |    |   |    value     |
|              |<--+    +--------------+    |   +--------------+    |   +--------------+
+--------------+   |    |   le_next    |----+   |   le_next    |----+   |   le_next    |-----+
                   |    |              |<--+    |              |<--+    |              |     |
                   |    +--------------+   |    +--------------+   |    +--------------+   -----
                   +----|   le_prev    |   +----|   le_prev    |   +----|   le_prev    |    ---
                        +--------------+        +--------------+        +--------------+     -
```

## 双向链表的访问

### 双向链表的访问宏

``` cpp
#define LIST_FIRST(head)        ((head)->lh_first)
#define LIST_END(head)          NULL
#define LIST_EMPTY(head)        (LIST_FIRST(head) == LIST_END(head))
#define LIST_NEXT(elm, field)   ((elm)->field.le_next)

#define LIST_FOREACH(var, head, field)              \
    for ((var) = LIST_FIRST(head); (var)!= LIST_END(head); (var) = LIST_NEXT(var, field))
```

### 双向链表的访问宏的作用

从宏定义名即可看出每个宏的作用：

- `LIST_FIRST`：获取双向链表的首节点
- `LIST_END`：双向链表的末尾，值恒为 `NULL`
- `LIST_EMPTY`：判断双向链表是否为空，为空时返回真
- `LIST_NEXT`：获取双向链表的下一个节点的地址
- `LIST_FOREACH`：遍历双向链表

> 注意：在使用 `LIST_FOREACH` 遍历双向链表时要避免链表的初始化、删除、替换操作！

### 双向链表的访问过程

#### 双向链表的遍历

`libevent` 的双向链表只支持从首节点向尾节点遍历。

要想完成遍历操作，有三个关键点：遍历的起始节点，遍历的下一节点的获取，遍历的结束条件。

通过以上对双向链表结构的分析轻易得出以上内容，即可完成链表的遍历操作：

- 遍历的起始节点：可以通过头节点指向的地址获取
- 遍历的下一节点的获取：可以通过当前节点的 `le_next` 元素指向的地址获取
- 遍历的结束条件：`le_next` 元素指向的地址为空

## 双向链表操作

### 初始化

#### 初始化宏

``` cpp
#define LIST_INIT(head) do {                        \
    LIST_FIRST(head) = LIST_END(head);              \
} while (0)
```

### 初始化过程

链表的初始化实际上只是初始化了头节点，由于头节点的 `lh_first` 与首节点相连，而此时链表为空，因此将头节点的 `lh_first` 置为 `NULL`。

### 头插法

#### 头插法宏

``` cpp
#define LIST_INSERT_HEAD(head, elm, field) do {     \
    if (((elm)->field.le_next = (head)->lh_first) != NULL)              \
        (head)->lh_first->field.le_prev = &(elm)->field.le_next;        \
    (head)->lh_first = (elm);                       \
    (elm)->field.le_prev = &(head)->lh_first;       \
} while (0)
```

#### 头插法步骤

头插法适用于已知头节点时，在首节点前插入新节点，不用关心首节点的有无，操作步骤如下：

1. 将新节点的 `next` 指向原来的首节点
2. 将原来的首节点的 prev 从指向 first 改为指向新节点的 `next` 指针
3. 将 `first` 指针从指向原来首节点改为指向新节点
4. 将新节点的 `prev` 指针指向 `first`

注意：

* 必须保证步骤 1 在步骤 3 之前。

### 前插法

#### 前插法宏

``` cpp
#define LIST_INSERT_BEFORE(listelm, elm, field) do {                    \
    (elm)->field.le_prev = (listelm)->field.le_prev;                    \
    (elm)->field.le_next = (listelm);               \
    *(listelm)->field.le_prev = (elm);              \
    (listelm)->field.le_prev = &(elm)->field.le_next;                   \
} while (0)
```

#### 前插法步骤

前插法适用于已知非头节点时，在已知节点前插入新节点，不用关心头节点是否已知，操作步骤如下：

1. 将新节点的 `prev` 指向已知节点的 `prev` 指向的位置
2. 将新节点的 `next` 指向已知节点的地址
3. 将已知节点的 `prev` 指向的地址的指向改为新节点的地址
4. 将已知节点的 `prev` 指向的地址指向新节点的 `next` 指针

注意：

* 必须保证步骤 1 在步骤 3 之前，必须保证步骤 3 在步骤 4 之前。

### 后插法

#### 后插法宏

``` cpp
#define LIST_INSERT_AFTER(listelm, elm, field) do {                     \
    if (((elm)->field.le_next = (listelm)->field.le_next) != NULL)      \
        (listelm)->field.le_next->field.le_prev = &(elm)->field.le_next;\
    (listelm)->field.le_next = (elm);               \
    (elm)->field.le_prev = &(listelm)->field.le_next;                   \
} while (0)
```

#### 后插法步骤

前插法适用于已知非头节点时，在已知节点后插入新节点，操作步骤如下：

1. 将新节点的 `next` 指向已知节点的 `next` 指向的位置
2. 将已知节点的 `next` 指向的节点的 prev 指向新节点的 `next` 元素地址
3. 将已知节点的 `next` 指向新节点的地址
4. 将新节点的 `prev` 指向的地址指向已知节点的 `next` 指针

注意：

* 必须保证步骤 1 或步骤 2 在步骤 3 之前，因为要保证能找到已知节点的后一个节点。

### 删除节点

#### 删除节点宏

``` cpp
#define LIST_REMOVE(elm, field) do {                \
    if ((elm)->field.le_next != NULL)               \
        (elm)->field.le_next->field.le_prev = (elm)->field.le_prev;     \
    *(elm)->field.le_prev = (elm)->field.le_next;   \
} while (0)
```

#### 删除节点步骤

删除节点只需要对待删除节点的前后（可能包括头节点）节点进行操作：

1. 如果存在后继节点，则将后继节点的 `prev` 赋值为待删除节点的 `prev`
2. 将待删除节点的 `prev` 指向的指针赋值为待删除节点的 `next`

> 注意：libevent 中实现的双向链表，删除操作不需要知道表头信息，仅需要直到待删除节点信息

### 替换节点

#### 替换节点宏

``` cpp
#define LIST_REPLACE(elm, elm2, field) do {         \
    if (((elm2)->field.le_next = (elm)->field.le_next) != NULL)         \
        (elm2)->field.le_next->field.le_prev = &(elm2)->field.le_next;  \
    (elm2)->field.le_prev = (elm)->field.le_prev;   \
    *(elm2)->field.le_prev = (elm2);                \
} while (0)
```

#### 替换节点步骤

1. 将新节点的 `next` 指向旧节点的 `next` 指向的位置
2. 将旧节点的 `next` 指向的节点的 `prev` 指向新节点指 next 地址
3. 将新节点的 `prev` 指向旧节点的 `prev` 指向的位置
4. 将新节点的 `prev` 指向的地址指向新节点地址

> 注意：libevent 中实现的双向链表，替换操作不需要知道表头信息，仅需要直到待替换节点信息和新节点信息

## 拓展问题

### Q1

Q1：节点的 `le_prev` 指针为什么不指向前一节点的首地址，而是指向前一节点的 `le_next` 指针或头节点的 `lh_first` 指针

A1：节点的 `le_prev` 指针为一个二级之指针，其指向前一节点的 `le_next` 指针或头节点的 `lh_first` 指针的原因有两点：

1. 节点中存在一个和其余节点结构不一致的头节点，`le_prev` 在指向头节点时无法指向头节点的首地址，否则在头节点中包含其他结构元素时，增加了寻找头节点中 `lh_first` 难度，故直接指向头节点中 `lh_first` 所在地址。

2. 如果节点的 `le_prev` 指向前一节点的首地址，在进行前插法、删除节点、替换节点时就必须知道头节点的地址，因为前插法在首节点前插入、删除首节点、替换首节点时需要修改头节点的 lh_first 指针，这样增加了参数的传递个数。而指向前一节点的 `le_next` 指针或头节点的 `lh_first` 指针在进行以上操作时，只需要简单地修改 `le_prev` 指针指向位置的值即可，大大降低使用以上接口进行链表操作的难度。

## 参考资料

* [1] [libevent-2.1.11-stable.tar.gz (Released 2019-08-01)](https://github.com/libevent/libevent/releases/download/release-2.1.11-stable/libevent-2.1.11-stable.tar.gz)
