---
layout: post
title: 游客模式
subtitle: 借助这个需求引入 APT 和 ASM 的使用，值得吹一番
image: /img/lifedoc/jijiji.jpg
tags: [周报]
---

sheet

| 家长控制功能 | 使用了 APT |
|---|---|
| 改功能影响较大，业务复杂 | 使用了 APT 技术去 hook 很多可能会弹出的页面，通过 annotation 配合 APT 可以简单的去定义如何弹窗以及弹窗样式等等 |
| 具体 APT 用法 | 定义注解有几个参数，是否为首页，弹窗位置，下部弹出还是中间弹出，使用 APT 注解解释器去扫描对应的类有这个注解的就去通过 ASM 在 onResume 或者 具体地方插入具体显示控制的方法调用 |
| 具体 ASM 用法 | 定义了常用显示方法，在编译期间插入对应的代码到 onResume 阶段，判断是游客的话就弹窗 |
| 遇到的问题 | APT 和 ASM 是第一次特别是 ASM ，主要是不了解字节码的插入方式，解决方式通过 ASM Plugin 反编译代码看看正常的 ASM 代码应该怎样写 |

### 类图

![](https://raw.githubusercontent.com/XPJ1993/images/master/PGC类图.png)


