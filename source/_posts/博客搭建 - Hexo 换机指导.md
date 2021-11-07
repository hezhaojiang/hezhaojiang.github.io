---
title: 博客搭建 - Hexo 换机指导
categories:
  - 博客搭建
tags:
  - Hexo
abbrlink: 6a28acc4
date: 2020-12-20 08:22:19
---
由于最近更换电脑，需要将旧电脑的一些资料转移至新电脑，其中 `Hexo` 的资料转移过程中踩了不少坑，所以在此记录以下，防止后续换机时继续踩坑。

<!-- more -->

## 安装 Hexo 环境

首先需要在新电脑上安装 `Hexo` 环境

### 安装 Node.js

这里是第一个坑，安装的 `Node.js` 建议版本为 12.14，如果安装的版本过高，后续 `hexo` 命令执行时可能会出现以下错误：

``` node
(node:2396) Warning: Accessing non-existent property 'lineno' of module exports inside circular dependency
(Use `node --trace-warnings ...` to show where the warning was created)
(node:2396) Warning: Accessing non-existent property 'column' of module exports inside circular dependency
(node:2396) Warning: Accessing non-existent property 'filename' of module exports inside circular dependency
(node:2396) Warning: Accessing non-existent property 'lineno' of module exports inside circular dependency
(node:2396) Warning: Accessing non-existent property 'column' of module exports inside circular dependency
(node:2396) Warning: Accessing non-existent property 'filename' of module exports inside circular dependency
```

### 安装 Hexo

安装好 `Node.js` 后直接安装 `Hexo`，安装命令如下：

``` bash
npm install hexo-cli -g
npm install hexo --save
```

安装完发现无法运行 `Hexo` 命令，需要进行如下设置：

1. 以管理员身份运行 `powershell`

2. 执行 `set-executionpolicy remotesigned`

## 拷贝 Hexo 资料文件夹

拷贝除 .deploy_git 和 public 文件夹以外的所有内容

![Hexo文件夹](https://gitee.com/hezhaojiang/MyPics/raw/master/img/20201220200545.png)

此时即可用 `hexo g` 等命令正常使用 `Hexo` 了~~
