---
title: 善用佳软 - Markdown 工具之 Mermaid
categories:
  - 工具软件
tags:
  - Markdown
abbrlink: 4689dd6d
date: 2020-10-14 15:11:00
---
`Mermaid` 是一个用于画流程图、状态图、时序图、甘特图的库，使用 `JS` 进行本地渲染，广泛集成于许多 Markdown 编辑器中。

`Mermaid` 作为一个使用 `JS` 渲染的库，生成的不是一张图片，而是一段 `HTML` 代码，因此安全许多。

官网：<https://mermaidjs.github.io/>
Github 项目地址：<https://github.com/knsv/mermaid>

<!--more-->

## Mermaid 流程图

流程图，英文 `Flow Chart`，是使用图形表示算法思路是一种方法。在 `Hexo` 中提供了插件，使得我们可以利用 `Mermaid` 这个库来实现流程图，它的显示结果非常漂亮优雅。

`Mermaid` 提供的流程图是一个简化版的，它主要由节点和连线组成，再以文字辅助，总共三种要素。

这里的三、四、五，其中三表示三种基本线型，四表示流程图四个方向，五表示五种节点形状。

### 三种基本线型及扩展

#### 三种基本线型及带箭头的基本线型

节点之间通过连线来连接，一共有 3 种线的形状，虚线：`-.-`、实线：`–--`、粗实线:`===`。

在基本线型符号串的右边加上 `>` 符号，去掉左边第 1 个符号，使其保持仍然是 3 个符号，就获得了带箭头的线型。虚线有点特殊，它的第一个符号可以不去掉。

``` node
%% 代码块类型 mermaid
graph TB
A1 -.- B1
A2 --- B2
A3 === B3
A4 .-> B4
A5 --> B5
A6 ==> B6
```

效果如下：

``` mermaid
%% 代码块类型 mermaid
graph TB
A1 -.- B1
A2 --- B2
A3 === B3
A4 .-> B4
A5 --> B5
A6 ==> B6
```

#### 在线段中间加上说明文字

上面 6 种连线，中间都可以加上说明文字。带文字连线的符号串是这样来的：

- 在符号串的左边加上说明文字
- 再在说明文字的左边加上基本线型的符号串的前两个符号

``` node
%% 代码块类型 mermaid
graph TB
A1 -. 虚线文字 -.- B1
A2 -. 虚线文字 .-> B2
A3 -- 实线文字 --- B3
A4 -- 实线文字 --> B4
A5 == 粗线文字 === B5
A6 == 粗线文字 ==> B6
```

效果如下：

``` mermaid
%% 代码块类型 mermaid
graph TB
A1 -. 虚线文字 -.- B1
A2 -. 虚线文字 .-> B2
A3 -- 实线文字 --- B3
A4 -- 实线文字 --> B4
A5 == 粗线文字 === B5
A6 == 粗线文字 ==> B6
```

### 四种图形走向

所有节点之间的连线，带箭头的表示其具有方向性，一块 `Mermaid` 代码块内，所有节点连线方形是一致的。那么总共有如下四种，用相应的两个字符表示。

- `TB`: 从上到下（也可以用 `TD`）
- `BT`: 从下到上
- `LR`: 从左到右
- `RL`: 从右到左

其中 `T` 表示 `top`，`B` 表示 `bottom`，`L` 表示 `left`，`R` 表示 `right`。这里又有个特殊的，`TB` 也可用 `TD` 表示。

下面的图分别表示上面四种图形走向：

``` node
%% 代码块类型 mermaid
graph LR
A1 -. 虚线文字 .-> B1
```

如 `LR` 图形走向效果如下：

``` mermaid
%% 代码块类型 mermaid
graph LR
A1 -. 虚线文字 .-> B1
```

### 五种节点外观

`Mermaid` 只提供了 5 种节点的外观。一般就可以满足要求。

| 语法 | 形状 | 示例 |
| - | - | - |
| [] | 矩形 | 矩形示例 |
| () | 圆角矩形 | 圆角矩形示例 |
| (()) | 圆 | 圆示例 |
| {} | 菱形 | 菱形示例 |
| >] | 三角形矩形组合 | 三角矩形示例 |

> 为什么菱形不是 `>`？ 因为这个符号要拿来表示箭头，分身乏术，只能用 `{}` 来表示。

``` node
%% 代码块类型 mermaid
graph TB
A1[矩形 A1]      -. 虚线文字 -.- B1[矩形 B1]
A2(圆角矩形 A2) -. 虚线文字 .-> B2(圆角矩形 B2)
A3((圆 A3))     -- 实线文字 --- B3((圆 B3))
A4{菱形 A4}     -- 实线文字 --> B4{菱形 B4}
A5 >三角矩形 A5] == 粗线文字 === B5 >三角矩形 B5]
```

效果如下：

``` mermaid
%% 代码块类型 mermaid
graph TB
A1[矩形 A1] -. 虚线文字 -.- B1[矩形 B1]
A2(圆角矩形 A2) -. 虚线文字 .-> B2(圆角矩形 B2)
A3((圆 A3)) -- 实线文字 --- B3((圆 B3))
A4{菱形 A4} -- 实线文字 --> B4{菱形 B4}
A5>三角矩形 A5] == 粗线文字 === B5>三角矩形 B5]
```

#### 一种子图形

`Mermaid` 还提供了子图形，可以被嵌入图形中。子图形以 `subgraph` 开始，以 `end` 结束。子图形必须提供标题，不能对图形方向作主。

``` node
%% 代码块类型 mermaid
graph TB
A --> D
A --> F
subgraph T2
C --> D
C --> subName1((sub))
end
subgraph T1
A --- B
end
subgraph T3
E -.- F
end
```

效果如下：

``` mermaid
%% 代码块类型 mermaid
graph TB
A --> D
A --> F
subgraph T2
C --> D
C --> subName1((sub))
end
subgraph T1
A --- B
end
subgraph T3
E -.- F
end
```

## Mermaid 时序图

时序图用来表示几个交互对象之间传递消息，处理事务的时间顺序的过程。核心要素是参与者（或角色），消息传递的发起及接收者，角色处理自身任务。消息传递及角色任务在时间上先后排列。时序图上的时间不要求特别间却，只要体现先后顺序即可。

Mermaid 实现的时序图中，角色水平排列，从上向下表示时间发展的顺序。

### 定义角色

语法很简单

``` node
    participant 甲
    participant 乙
```

其中 `participant` 表示定义参与者，` 甲 ` 和 ` 乙 ` 是就是角色名，可以是字符串，不含空格。如果有需要，可以用 `as` 来增加角色的别名。

``` node
participant 成吉思汗 as 甲
```

需要注意：

- 定义参与者不是必须的，参与者按照出场先后顺序排列，如果要把某个角色排在前面，那么就一定要定义它。
- 预先定义的角色排列完毕之后。没有定义的统统按出场顺序跟在后面。
- 定义语句不一定要放在消息传递之前，在中间或者后面也是可以的。
- `Mermaid` 这里存在一个 bug。== 角色名后面一定不能有空格。空格和角色名一起被视作另外一个角色 ==

### 消息连线

通过连线来表示角色传递消息的过程。消息连线有虚线和实线两种基本线型，另外有增加箭头或者终止符的变化。

1. 基本线型
    - 实线 `->`
    - 虚线 `-->`
2. 箭头连线
    - 实箭头线 `->>`
    - 虚箭头线 `-->>`
3. 带终止符的连线
    - 实终止符线 `-x`
    - 虚终止符线 `--x`

下面我们让甲乙来演示一下综合效果

``` node
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
甲 -> 乙 : 你还好吗？
乙 --> 甲 : 你看我这不挺好吗？
甲 ->> 乙 : 来个箭头试试
乙 -->> 甲 : 来就来，我也会
甲 -x 乙 : 加个 x 试试？
乙 --x 甲 : 试试就试试，谁怕谁？
```

效果如下：

``` mermaid
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
甲 -> 乙 : 你还好吗？
乙 --> 甲 : 你看我这不挺好吗？
甲 ->> 乙 : 来个箭头试试
乙 -->> 甲 : 来就来，我也会
甲 -x 乙 : 加个 x 试试？
乙 --x 甲 : 试试就试试，谁怕谁？
```

### 角色的内部任务

任务处理过程，角色本身不仅传递消息，自身还存在任务，需要告诉它开始处理及处理完自己的任务。

开始处理用语句：`activate` 角色；处理完毕用语句：`deactivate` 角色。

多数情况下，都是角色在接收到消息时才会启动或者结束自身任务，因此偷懒用了个快捷的办法，在消息连线后面加上 `+/-`，表示接收消息的角色这时候应该开始或者结束自身任务处理。

如果不是上面这种情况，是需要用 `activate` 和 `deactivate` 语句来启动和结束任务的。

另外，时序图里的角色不能处理多任务。

如果有多个启动和完成任务的指令，记住他们一定是从最内侧进行配对的，下面举例说明。

``` node
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
乙 --> 甲: 甲启动任务 1
甲 ->> 乙: 空白
activate 甲
甲 -->> 乙: 上一句让甲启动了任务 2。
乙 -->> 甲: 甲结束了任务 1，只能匹配任务 2
甲 -x 乙: 这里有个终止符
deactivate 甲
乙 --x 甲: 上一句甲结束了任务 2，只能匹配任务 1
```

效果如下：

``` mermaid
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
乙 --> 甲: 甲启动任务 1
甲 ->> 乙: 空白
activate 甲
甲 -->> 乙: 上一句让甲启动了任务 2。
乙 -->> 甲: 甲结束了任务 1，只能匹配任务 2
甲 -x 乙: 这里有个终止符
deactivate 甲
乙 --x 甲: 上一句甲结束了任务 2，只能匹配任务 1
```

### 给角色贴身便签

给角色贴上便签 `note`，可以使其被更好地理解。便签的位置有角色的左边、右边或者上方，记住没有下方。

    Note left of | right of | over John,Alice: Text in note

上面的语法很好理解，Note 是固定词汇，`left of` | `right of` | `over` 是便签的位置，左、右或中，`John,Alice` 是被贴便签的角色，最后面的是便签内容。

> 注意： `over` 不带 `of` 且可以同时为多个角色贴便签。

代码如下

``` node
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
Note over 甲,乙 : 你们两都挺好的
甲 -->> 乙 : 你好
乙 ->> 甲 : 你也好
Note over 乙 : 你们两都挺好的
```

效果如下：

``` mermaid
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
Note over 甲,乙 : 你们两都挺好的
甲 -->> 乙 : 你好
乙 ->> 甲 : 你也好
Note over 乙 : 你们两都挺好的
```

### IF-ELSE 时序

时序在处理的时候，肯定会遇上有条件执行的情况，它的语法如下

``` node
alt 可选语句说明
...
else
...
end
```

其中的 `alt` 是 `alternative` 的缩写，即这有两组时序是二选一的，相当于 `if-else` 语句。

如果没有 `else`，那么用另外一个关键词 `opt`，他是 `optional` 的缩写。

``` node
opt 条件说明
end
```

``` node
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
甲 ->> 乙 : 你好吗？
alt 如果乙很好
乙 -->> 甲 : 我很好
else
乙 -->> 甲 : 我不好
end
甲->>乙 : 你有空吗？咱看电影去。
opt 乙有空
乙 -->> 甲 : 去吧
end
```

效果如下：

``` mermaid
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
甲 ->> 乙 : 你好吗？
alt 如果乙很好
乙 -->> 甲 : 我很好
else
乙 -->> 甲 : 我不好
end
甲->>乙 : 你有空吗？咱看电影去。
opt 乙有空
乙 -->> 甲 : 去吧
end
```

### 循环时序

把几个语句用 `loop end` 语句圈起来，这就是循环时序。语法如下

``` node
loop 循环语句说明
... 语句
end
```

示例：

``` node
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
甲 ->> 乙 : 有人在吗
乙 ->> 甲 : 在啊
loop 这俩话唠
甲 -->> 乙 : 你好
乙 -->> 甲 : 你好
end
甲 ->> 乙 : 我不玩了，只会说你好
```

效果如图：

``` mermaid
%% 代码块类型为 mermaid
sequenceDiagram
participant 甲
participant 乙
甲 ->> 乙 : 有人在吗
乙 ->> 甲 : 在啊
loop 这俩话唠
甲 -->> 乙 : 你好
乙 -->> 甲 : 你好
end
甲 ->> 乙 : 我不玩了，只会说你好
```

## Mermaid 甘特图

示例：

``` node
%% 代码块类型为 mermaid
gantt
dateFormat  MM-DD
title Shop项目交付计划

section 里程碑 1
数据库设计          :active,    p1, 08-15, 3d
详细设计            :           p2, after p1, 2d

section 里程碑 2
后端开发            :           p3, 08-22, 10d
前端开发            :           p4, 08-22, 8d

section 里程碑 3
功能测试            :           p6, after p3, 5d
上线               :           p7, after p6, 2d
交付               :           p8, after p7, 2d
```

效果如下：

``` mermaid
gantt
dateFormat  MM-DD
title Shop项目交付计划

section 里程碑 0.1
数据库设计          :active,    p1, 08-15, 3d
详细设计            :           p2, after p1, 2d

section 里程碑 0.2
后端开发            :           p3, 08-22, 10d
前端开发            :           p4, 08-22, 8d

section 里程碑 0.3
功能测试            :        p6, after p3, 5d
上线               :       p7, after p6, 2d
交付               :       p8, after p7, 2d
```

## 参考资料

* [1] [Hexo+Mermaid (一)：记住三、四、五，玩转 Mermaid 流程图 | 走进 IT](http://www.guide2it.com/post/2019-03-10-1-make-flowcharts-with-mermaid-in-markdown/)
* [2] [Hexo+Mermaid (二)：六步玩转 Mermaid 时序图 | 走进 IT](http://www.guide2it.com/post/2019-03-10-2-make-sequence-diagrams-with-mermaid-in-markdown/)
* [3] [Hexo+Mermaid (三)：用 Mermaid 在 Hexo 中实现超简单的甘特图 | 走进 IT](http://www.guide2it.com/post/2019-03-10-3-make-gantt-with-mermaid-in-markdown/)
