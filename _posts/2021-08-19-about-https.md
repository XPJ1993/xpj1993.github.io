---
layout: post
title: Https的理解
image: /img/lifedoc/jijiji.jpg
tags: [网络]
---

https的整体握手建立连接的过程，我们可以看到与Http最大的区别就是中间多了SSL/TLS的握手部分。

![](https://raw.githubusercontent.com/Pjex/images/master/https_handshake.png)


关于Https不可不知的知识点。
![](https://raw.githubusercontent.com/Pjex/images/master/20210819184707.png)

实际上，Https就是比Http多了SSL/TLS层，而在这一层做了很多辅助工作，例如CA证书，对称非对称加解密，多了TLS的多次握手等等。

