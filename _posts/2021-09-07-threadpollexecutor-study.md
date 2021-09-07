---
layout: post
title: Java 线程池学习
subtitle: 线程池是 Java 编程中非常好用的一个工具，对于多线程编程，这里简直是神器
image: /img/lifedoc/jijiji.jpg
tags: [Java]
---

Java 线程池学习

**线程池创建方式：**

| 创建线程池 | 创建方式 |
|---|---|
| 通过 Executors | newFixedThreadPool(int count) 创建一个核心线程和最多线程数量相同的线程池 |
| 同上 | newWorkStealingPool(int parallelism) 创建一个支持窃取任务的线程池 |
| 同上 | newSingleThreadExecutor() 创建一个单线程线程池 |
| 同上 | newCachedThreadPool 创建一个即时使用 1 分钟之后销毁多于线程的线程池，值得注意的这个线程池是没有核心线程的，一般处理瞬时任务量很大，干完这一票就没有的功能 |
| 同上 | newSingleThreadScheduledExecutor() 支持按照一定规则的创建 task 的线程池，可以 delay 一个 task |
| 同上 | newScheduledThreadPool(int size) 创建一个支持 delay task 的线程池，核心线程和最大线程数量一样 |
| new ThreadPoolExecutor() | 通过 new 基本的 ThreadPoolExecutor 去创建线程池，其中有几个参数需要知道他们的含义 |
| 创建自定义继承 ThreadPoolExecutor 的类 | 继承 ThreadPoolExecutor 通过这个可以控制更多的东西，例如 task 执行前和执行后都会有回调，我们可以在这个回调里面去做一些自定义的工作，开始加戏以及结束收尾 |

**线程池参数：**

| 线程池参数/方法 | 参数释义 |
|---|---|
| corePoolSize | 核心线程数 |
| maximumPoolSize | 最大线程数，用完之后会销毁释放资源的线程 |
| keepAliveTime | 非核心线程空闲多久释放线程 |
| TimeUnit |  时间单位 |
| BlockingQueue<Runnable> | 阻塞队列，当核心线程满载就会往这个队列添加，满了之后再开启非核心线程，最后在满了就会触发拒绝策略 |
| ThreadFactory | 线程构造工厂接口类，负责生成对应的线程 |
| RejectedExecutionHandler |  拒绝策略执行触发类，这里可以自定义实现在拒绝的时候再把这个 task （Runnable） 添加回去构造一个不丢任务的线程池，并且队列长度有限不浪费资源 |
| beforeExecute | 参数为 Thread t，Runnable r 运行一个 task 的时候提前通知，**这个是方便在执行一个 task 的时候给这个 task 增加一些戏，例如装饰器加东西** |
| afterExecute | 参数为 Thread t，Runnable r 运行一个 task 的时候运行完之后的后置提醒**这个是在执行完成后方便借助这个做一个 task 收尾的工作** |
| terminated | 线程池终止之后的回调**整体线程池的终止回调，可以通过这个去回收共用的工具或者连接池等等** |

### 线程池原理

### 线程池使用场景

### 线程池总结

