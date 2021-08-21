---
layout: post
title: TCP 连接与断开
subtitle: 三次握手与四次挥手
image: /img/lifedoc/jijiji.jpg
tags: [网络]
---

**三次握手**：

![](https://raw.githubusercontent.com/XPJ1993/images/master/tcp-handshake.gif)

为什么要三次握手，因为这样才能确保真正的建立了连接，例如第一次client向server发syn的header的时候就认为建立连接，实际上这消息服务端还没有收到呢，那么等server根据这个syn给client提供一个syn-ack的时候，我们能不能认为连接上了呢，毕竟服务器都说他收到了，不能，因为这会client断线了。。只有client再次跟服务器发送一个ack这时服务器才会给他分配socket提供服务。

这三次交互都在规定时间完成了，那么就可以认为建立连接了。

关于三次握手，咱在做对讲的时候也弄过一个三次握手协议，只是不是为了建立连接，而是为了解决重复弹窗，第一次就弹的话会有误发送的情况，或者这个push是一个延迟的信息，那么我再去询问一下这个是否在有效期，在的话我再弹窗，也是一个变形的三次握手吧，只不过我的第一次是push给我的。


**四次挥手**：

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210821110708.png)

第一次client跟server发送带有fin的tcp头，服务端解析后返回对应的tcp ack头，然后等数据发送完成后（或者直接断掉），server在向client发送fin，这时client收到之后回复一个ack，这时候才能真正的释放socket资源。

同样滴，如果直接第一次就断开呢，那么服务端都不知道你不用了，会导致服务端迟迟不释放socket资源，如果server发了ack就认为断了呢，还有数据没有发送完成么，只能等server去发送fin证明我的数据也发送完成了，可以结束了，客户端确认我收完了，咱们拜拜吧，这才能搞完。



