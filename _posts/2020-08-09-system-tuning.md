---
layout: post
title: 性能优化记录
categories: system
description: 本文主要介绍性能优化需关注的点
keywords: linux,tcp/ip,system tuning
---

## 基础设施优化
### cpu多级缓存
![cpu cache](/images/posts/system/cpu-cache.jpg) 

CPU 访问一次内存通常需要 100 个时钟周期以上，而访问一级缓存只需要 4~5 个时钟周期，二级缓存大约 12 个时钟周期，三级缓存大约 30 个时钟周期（对于 2GHZ 主频的 CPU 来说，一个时钟周期是 0.5 纳秒)。
#### 提升数据缓存的命中率
* 遇到这种遍历访问数组的情况时，按照内存布局顺序访问将会带来很大的性能提升
![copyij](/images/posts/system/copyij.png)
#### 提升指令缓存的命中率
##### CPU含有分支预测器

如果 if 中的条件表达式判断为“真”的概率非常高，我们可以用 likely 宏把它括在里面，反之则可以用 unlikely 宏

```cpp
#define likely(x) __builtin_expect(!!(x), 1) 
#define unlikely(x) __builtin_expect(!!(x), 0)
if (likely(a == 1)) …
```
#### 提升多核CPU下的缓存命中率

Linux 上提供了 sched_setaffinity 方法实现这一功能，其他操作系统也有类似功能的 API 可用，当多线程同时执行密集计算，且 CPU 缓存命中率很高时，如果将每个线程分别绑定在不同的 CPU 核心上，性能便会获得非常可观的提升。Perf 工具也提供了 cpu-migrations 事件，它可以显示进程从不同的 CPU 核心上迁移的次数。
#### 总结
* ***核心为提升缓存命中率***

### 内存池

### 索引

### 零拷贝

### 协程

### 锁

## 网络层优化

## 应用编解码层优化

## 总结

## 参考
* [系统调优必知必会](https://time.geekbang.org/column/intro/308)