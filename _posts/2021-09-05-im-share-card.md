---
layout: post
title: IM 卡片分享
subtitle: 主要是通过这个项目完成了内存泄漏治理，完成了优秀协议治理，完成了优秀框架制作
image: /img/lifedoc/jijiji.jpg
tags: [周报]
---

### 类图

![](https://raw.githubusercontent.com/XPJ1993/images/master/IMShareMain.png)

![](https://raw.githubusercontent.com/XPJ1993/images/master/IMBridgeMain.png)


### Star 原则总结

situation情景：

1. 做 IM 分享卡片发现 IM 有 anr 存在；
2. 代码比较老旧且角色划分极为不合理；
3. 代码很多处于中台，无法做大改。

task：

1. 找到 anr 根源，有四个，一个是主线程读取数据库，二个是不合理的全量刷新机制（有数据变动会清理 list 并重新刷新 UI ，假如有 1w 条也是这么个流程），三创建对应 chat 的时候在构造函数中就会拉取信息（本意是为下次加载快速，但是设计很垃圾），四一个类干了所有事，导致三也不能直接动，因为收发存啥的都是它干，基本上牵一发动全身；
2. 逐个击破，设计合理架构解决这个棘手问题，主线程必须 ko 借助协程使用 io 线程拉取，二，细粒度更新，改变全量更新机制，三，懒加载，按需加载，重载构造函数，四，增加桥接模式，找到手术切入点，将代码做最小化侵入，屏蔽实现，不把业务代码放到中台库；

action：

1. 分轻重缓急，画出架构图，类图；
2. 设计架构，完善架构，完成在刀尖上跳舞。

result：

1. 学生会转化率提升 50%；
2. anr 贡献降低 千分之一；
3. 代码无重大侵入，兄弟业务无感知。

**对数据敏感**

对数据敏感，在意所做功能对数据的影响，善于总结复盘功能得失，避免下次再犯。

