---
layout: post
title: synchronized
subtitle: ReentrantLock 区别
image: /img/lifedoc/jijiji.jpg
tags: [Java]
---

sheet

| synchronized | ReentrantLock |
|---|---|
| 对象头里面表示锁 | 上层 API 底层使用 CAS |
| 不可重入 | 可重入 |
| 不公平 | 在构造的时候可以指定是否是公平锁 |
| 控制粒度无法狠详细 | 可以使用 condition 精确控制每一个线程 |
| JVM 层面底层是 monitor enter 和 monitor exit | 底层基于 CAS |
| 不可中断 | 可以中断 |
| 无需释放 | 手动释放，一般在 finally 里面去释放 |

接下来要分析 condition 和 ReentrantLock 以及 它父类的关系。

