---
layout: post
title: 线程池深度探险（1）：前情提要
subtitle: 整体性学习比喻法，解释牛皮克拉斯的线程池
image: /img/lifedoc/jijiji.jpg
tags: [ThreadPool]
---

线程池值得一个 tag。


| 本篇主题 | 引入线程池类比工厂 |
|---|---|
| 使用整体学习法中的比喻方法介绍线程池和工厂的恩怨 | 线程池管理的 worker 就是工厂管理的工人，么么哒 |


### 线程池啊，你是一个工厂吗

关于线程池的想象，应用整体学习法的比喻方法。

线程池是虾米捏，线程池阔以类比为工厂，就像 Java 线程池里定义干活的人为 worker 一样，线程池就负责这些 worker 工作滴鸭。

拿最经典最完全滴线程池举例，他有几个可选参数，第一一个是<span style="color:#871F78;">核心线程数 coreThreadSize ，这玩意就是工厂里面的核心玩家</span>，哪怕暂时厂子接不到订单（runnable）了，也会养着他们；第二个参数是最大线程数，就是最多多少工人，这些超过核心线程数的线程是有可能被释放的，就是<span style="color:#871F78;">短期工</span>，订单充足核心干不完了上他们；之后的<span style="color:#871F78;">第三四参数是空闲时间以及时间单位</span>，这个呢就是说，咱厂子效益不是特别好的时候，单子核心工人就干得了，那么就把这些外包非核心先 fire 掉，省点资本了；第五个参数呢，是， blocking queue，线程安全滴阻塞队列，这个就是<span style="color:#871F78;">咱厂子最多可以承接的排队缓存单子</span>了，这些单子是咱滴核心工人全部在忙碌阶段，那么接了单先放着，紧接着咱们就启动外包人员；第六个参数是 reject handler，就是说虽然咱想挣钱，但是咱是信誉至上滴人，如果咱单子排队都排满了，<span style="color:#871F78;">那就拒单吧，不能耽误别人事哈</span>，只是拒单滴时候会跟人说，我拒单了，我拒的是那单，在这里如果人觉得这厂子靠谱还可以扔过来，在扔过来因为有些资源释放了，那还会给做单的。

哎呀，以厂子比喻，这线程池可太形象了，呦吼，金光闪闪的单子，那可是玛尼呀。

线程池工厂有好几种，上述就是咱经典厂子， executors 也提供了几种给咱创建工厂的方法，第一类工厂是 newSingleThreadPool 单线程，一工人工厂，这厂子就一个工人，有啥事都是它自己干，它就是唯一核心；第二类工厂是 newFixedThreadPool 传递的参数是一共几个工人，这几个工人都是核心，后面拒绝策略了，排队 queue 了，不传递都有默认值，例如 queue 默认就是无限的长度；第三类工厂 <span style="color:#871F78;">newScheduleFixThreadPool / single</span> 这类按规划跑的工厂，因为有的人说我的单子不是立马做，可以按计划来，都可以，比方说这单子我 delay 个十天半个月在搞，到时候你给我搞完就行了；第四类呢是 <span style="color:#871F78;">newStealenThreadPool</span> 这个工厂吊了，是 jdk  1.8 之后加的新型工厂，这工厂是会偷任务滴智能工厂，虽然叫偷，但是却明明白白的提现了这厂子里工人的主观能动性，我没活了，我去争取，我踏马去卷，找活干，这玩意底层是基于 <span style="color:#871F78;">fork join</span> 的实现。这个可以研究研究，同时研究下 kotlin 实现的那版线程池是杂么搞得，我看他也有 steal 这一步，貌似也没用 fork join 对比一下呗。

### 别的奇思妙想呀

另外有一个单线程很高效的做法是啥呢，是 epoll 配合单线程去调度，这个 Android 里面有用， Redis 也在管理 socket，但是听说 Redis 这玩意单线程搭配 epoll 是很高效滴，但是咱 Android 里的 HandlerThread 虽然也是 epoll 机制，但是感觉没有那么高效呢，原因是阻塞执行 message 吗，轮到谁了谁就阻塞执行这个 task 这时这个 task 可以独占这个线程。那么 Redis 是判断这个 task 需要等待就挂起了吗？如果是这样和协程类似，协程的高效也体现在这里，可以更智能的调度，使用挂起而非阻塞去搞事，事实上虽然是挂起，这事情也在干啊，例如接收网络 socket 的 io，或者读取磁盘 io，接收这活是谁做的呢，是系统线程还是另一个线程，在咱们线程挂起只是不消耗和阻塞咱了，这活也有人在做啊。

这玩意得学习下。

另外，kotlin 语言级别上给支持 select 还是大有可为滴，说不定以后 kotlin 了还支持 epoll 勒。

接下来写一系列线程池流程，从抛入 runnable 开始，讲解进厂历险记。

### io 多路复用

然后写 io 多路复用 相关的，第一个首属经典 select ，select 大佬功能如其名，就是突出一个 select 选择，select 大佬的实现是基于数组的，这个数组存起来那些 io 需要多路复用的使用者，这个数组长度为 1024，因此这大佬也限制了最多可以让这么多人复用，另外一个就是这大佬发现有 io ready 之后会遍历这个数组，效率也是有点底下；第二个就是 poll ，这个大佬修正了 select 大佬的数组长度限制，就转变了存储方式搞成了链表去存储，这时候长度是没有限制了，但是还是遍历去发现和通知，效率也就那样；第三个则是终极大佬， epoll ，epool 可了不得了，他一口气解决了上述俩人的缺点，epoll 大佬就突出一个厉害，我不限制长度，我通知效率贼高，不限制长度咱知道，链表存储被，通知效率提高咋搞的捏，原来呀，是 epoll 大佬聪明了，我不遍历了，我存起来一个 io 对应的 callback ，这个 io 上有 ready 信号了，我不遍历，直接 callback 伺候，效率可太高了。上边两个不够优秀， epoll 大佬这么优秀那么用法为啥子呢。

```c++
// epoll 用法
// 第一步，创建一个 epoll 实例并且返回其 fd
epoll_create(2)
// 第二步，注册对改 fd 感兴趣的文件描述符
epoll_ctl(2)
// 第三步，阻塞当前线程等待 epoll 的唤醒
epoll_wait(2)

// poll 用法
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events  监听fd 上哪些事件，它是一系列事件按位或 */
    short revents;    /* returned events 由内核修改，来通知应用程序fd 上实际上发生了哪些事件 */
};

// select 用法
// 每次都要重新设置nfds.因为select返回时，rfds被内核改变，里面只保存了就绪的文件描述符
/* nfds是监听文件描述符的总数。它通常被设置为select监听的所有文件描述符的最大值加1.
readfds, writefds, exceptfds指向可读、可写、异常等事件对应的文件描述符集合 */
int select(int nfds,fd_set * readfds,fd_set * writefds,fd_set * exceptfds,struct timeval * timeout);
// select https://www.cnblogs.com/zuofaqi/p/9622860.html
```


