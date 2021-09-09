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

#### 线程阻塞定义

线程阻塞的时候会让出 cpu 

理解二：阻塞（pend）就是任务释放CPU，其他任务可以运行，一般在等待某种资源或信号量的时候出现。挂起（suspend）不释放CPU，如果任务优先级高就永远轮不到其他任务运行，一般挂起用于程序调试中的条件中断，当出现某个条件的情况下挂起，然后进行单步调试

理解一：挂起是一种主动行为，因此恢复也应该要主动完成，而阻塞则是一种被动行为，是在等待事件或资源时任务的表现，你不知道他什么时候被阻塞(pend)，也就不能确切 的知道他什么时候恢复阻塞。而且挂起队列在操作系统里可以看成一个，而阻塞队列则是不同的事件或资源（如信号量）就有自己的队列

```java
/*
    挂起线程的意思就是你对主动对雇工说：“你睡觉去吧，用着你的时候我主动去叫你，然后接着干活”。
    
    使线程睡眠的意思就是你主动对雇工说：“你睡觉去吧，某时某刻过来报到，然后接着干活”。
    
    线程阻塞的意思就是，你突然发现，你的雇工不知道在什么时候没经过你允许，自己睡觉呢，但是你不能怪雇工，肯定你 这个雇主没注意，本来你让雇工扫地，结果扫帚被偷了或被邻居家借去了，你又没让雇工继续干别的活，他就只好睡觉了。至于扫帚回来后，雇工会不会知道，会不会继续干活，你不用担心，雇工一旦发现扫帚回来了，他就会自己去干活的。因为雇工受过良好的培训。这个培训机构就是操作系统。
*/
```

