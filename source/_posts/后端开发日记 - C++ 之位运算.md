---
title: 后端开发日记 - C++ 之位运算
categories:
  - 后端开发日记
tags:
  - C++
  - 数据结构与算法
abbrlink: aa6f0789
date: 2020-11-05 16:32:28
---
在 `LeetCode` 刷题时，经常会用到 `C++` 的一些位运算函数，在此记录下防止需要用时忘记。

> 需要注意，以下函数均为 `gcc` 提供的内置位运算函数，在其他编译环境上无法使用。

<!-- more -->

## __builtin_parity(n)

该函数是判断 `n` 的二进制中 `1` 的个数的奇偶性

``` cpp
int n = 15;                         // 二进制为 1111
int m = 7;                          // 二进制为 111
cout<<__builtin_parity(n)<<endl;    // 偶数个 1，输出 0
cout<<__builtin_parity(m)<<endl;    // 奇数个 1，输出 1
```

## __builtin_popcount(n)

该函数时判断 `n` 的二进制中有多少个 `1`

``` cpp
int n = 15;                         // 二进制为 1111
cout<<__builtin_popcount(n)<<endl;  // 输出结果为 4
```

## __builtin_ctz(n)

该函数判断 `n` 的二进制末尾后面 `0` 的个数，`n = 0` 时结果未定义

``` cpp
int n = 1;                      // 二进制为 1
int m = 8;                      // 二进制为 1000
cout<<__builtin_ctzll(n)<<endl; // 输出 0
cout<<__builtin_ctz(m)<<endl;   // 输出 3
```

## __builtin_clz(n)

`n` 前导 `0` 的个数, `n = 0` 时结果未定义

``` cpp
long long n = 1;                // 二进制为 000....001 64 位整数
int m = 8;                      // 二进制为 000...1000 32 位整数
cout<<__builtin_clzll(n)<<endl; // 输出 63
cout<<__builtin_clz(m)<<endl;   // 输出 28
```

## __builtin_ffs(n)

该函数判断 `n` 的二进制末尾最后一个 `1` 的位置，从 `1` 开始

``` cpp
int n = 1;                      // 二进制为 1
int m = 8;                      // 二进制为 1000
cout<<__builtin_ffs(n)<<endl;   // 输出 1
cout<<__builtin_ffs(m)<<endl;   // 输出 4
```
