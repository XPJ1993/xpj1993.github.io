---
layout: post
title: Kotlin 特性学习（5）通道和 Actor
subtitle: Kotlin 通道与 Actor
image: /img/lifedoc/jijiji.jpg
tags: [Kotlin]
---


### 脑图

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210829161538.png)

### 总结

Channel 可以用于协程间通信，如果构建一个不关闭的通道那就是管道，通道是基于协程的，取消的时候除了调用其自身的 cancel 还可以调用 coroutineContext cancelChildren。

通道目前看的作用就是协调各个协程之间的通信，有了这个就可以去做一些协调或者通信，通道支持一对多，多对一，多对多等，利用通道，可以实现协程之间的交叉打印这个需求，主要是因为对于通道来说协程是公平的，可以公平争抢协程产生的数据或者信号。

actor 目前还属于试验阶段，试用了一下 actor 结合 seal class 去处理协程并发，发现效率还不如一个粗粒度限制一个线程去操作效率高，有点尴尬。

可以结合他的思想去构建自己的协程并发框架。

更优化的可以搞协程之间去偷任务啥的，线程有个线程窃取，协程也可以，排队的任务多并且有别的协程空闲可以想办法把这些 task 给到空闲的协程，完成这个调度就会更加完美的压榨 CPU。
