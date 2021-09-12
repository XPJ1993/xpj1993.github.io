---
layout: post
title: parcent baby 项目
subtitle: 为孩子写的百分项目
image: /img/lifedoc/jijiji.jpg
tags: [Project]
---

别人都唱为你写诗，咱这做程序员的也没啥本事就为儿子写个 App 吧。

| module | 技术选型 |
|---|---|
| 数据库 | room |
| 音视频 | androidx.media |
| 首页 | FragmentActivity + Fragment |
| includeBuild | Gradle 复合构建，完成多版本统一依赖版本号控制 https://segmentfault.com/a/1190000010129596 |
| 上拉加载与下拉刷新 | 使用多 type 对应的 recyclerview 的框架，配合新增的自定义 adapter 完成下拉加载与下拉刷新的无感知 |
| content1 | content2 |

#### 20210912 第一阶段

完成了 百分宝贝 的第一阶段，首页框架初步完成，使用 假数据 去做的 UI 填充，下一步就是增加交互，增加数据库存储增加和减少分数的存储，另外需要设计和实现项目的增删改查的交互，例如单击 item 的时候可以选择更改此项的内容还是删除此条目。

| 待添加功能 | 涉及模块 |
|---|---|
| 条目修改 | 单击条目弹窗提示可以拥有的交互，例如删除、修改等 |
| 数据库设计 | 设计几个关键的 table |
| room 使用学习 | 学习 room 的创建与使用 例如增删改查 |
| 增加 viewmodel 完成对数据库的操作 | 在 viewmodel 里与数据库交互 |
| viewmodel 使用 flow 处理数据 | 在 viewmodel 中使用 flow 在 io 线程去操作数据的读取与转换，在 main 线程去更新 livedata 数据 |
| 使用 meituan 老哥的 livedatabus | 使用基于 livedatabus 的事件通知框架，正好更好的理解他的思想，或者自己直接把代码撸下来，在源码基础上做一些优化也是可以的，没看他就是使用装饰器去修改之前基于反射的实现吗，反射有两个问题一个是效率有轻微影响，再者是反射可能有些 api 谷歌会屏蔽 |
| content1 | content2 |
| content1 | content2 |

#### 下一版本规划

增加语音日记功能，增加语音导出功能（导出为 zip 文件，解压之后都是 mp3）

研究那个坚果云的备份机制是怎样的，尝试接入对应的 api 使用云备份本地数据。


