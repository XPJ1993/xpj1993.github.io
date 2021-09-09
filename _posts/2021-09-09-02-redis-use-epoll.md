---
layout: post
title: Redis epoll 使用方式
subtitle: redis 使用 epoll 机制在单线程下提供高并发的可能性
image: /img/lifedoc/jijiji.jpg
tags: [Kotlin]
---

redis 是怎么提高他的并发能力的，在 redis 的设计里，万物皆为事件，因为 redis 的数据都是内存读取的，而内存读取是非常快的，因此 redis 为了避免多线程操作数据带来的加锁成本就选择了单线程配合 epoll 提高事务处理能力，具体地就是每个请求过来的时候 redis 都会包装为基于 epoll 的事件，发出去之后就等着结果回来了，实际上发出去之后后续操作是多线程的，但是最终的数据都会回到这个单线程搭配 epoll 的线程上面。

#### kotlin suspend 关键字含义

A suspending function is simply a function that can be paused and resumed at a later time. They can execute a long running operation and wait for it to complete without blocking.

上面例子中的delay方法是一个suspend function.
delay()和Thread.sleep()的区别是: delay()方法可以在不阻塞线程的情况下延迟协程. (It doesn't block a thread, but only suspends the coroutine itself). 而Thread.sleep()则阻塞了当前线程.

所以, suspend的意思就是协程作用域被挂起了, 但是当前线程中协程作用域之外的代码不被阻塞.

