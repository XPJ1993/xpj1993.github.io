---
layout: post
title: Kotlin 特性学习（3）
subtitle: Kotlin 流学习
image: /img/lifedoc/jijiji.jpg
tags: [Kotlin]
---

开个坑，学习 kotlin 的 flow。

### flow 脑图

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210828174303.png)

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210828174431.png)


![](https://raw.githubusercontent.com/XPJ1993/images/master/20210828174516.png)


kotlin 的 flow 是基于协程的，是协程并发的一种，flow 是为了解决协程只发送一个值的问题，达到连续发射的目的，流的特点是拥有大量的操作符，流是连续的，经过了各种中间操作符的控制会达到得到最终结果的一种方式。

流和响应式编程特别相似，都是拥有大量的操作符，操作符有各种含义，方便对数据进行各种操作，并且流式编程是一种编程范式，一种编程思想，作为一个优秀的程序员，要及时跟得上技术的发展，保持对新技术的热情。


