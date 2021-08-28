---
layout: post
title: Kotlin 特性学习（4）
subtitle: Kotlin 协程异常及监督机制
image: /img/lifedoc/jijiji.jpg
tags: [Kotlin]
---

### 脑图

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210828175807.png)

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210828175914.png)

### 总结

**异常机制**：

协程的异常是透明的，异常可以透传到外部，可以在外部进行 try catch 获取协程中产生的异常，也可以通过 **.catch{}** 操作符获取协程中的异常

**监督机制**：

协程中有 SuperVisorJob 以及 SuperVisorScope 两个限定监督作业和监督作用域，监督的特色就是监督中的协程之间不会因为异常而受到彼此影响，也不会因为异常而影响父协程
