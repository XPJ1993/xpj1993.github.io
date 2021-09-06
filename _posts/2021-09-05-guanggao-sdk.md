---
layout: post
title: 广告 SDK 架构图
subtitle: 广告 SDK 的架构图解释与梳理
image: /img/lifedoc/jijiji.jpg
tags: [周报]
---

sheet

| 分层结构 | 分层作用 |
|---|---|
| 广告 SDK 使用经典的分层结构 | 分层的最大作用就是层次之间职责清晰，可替代性很强，随随便便换个组件啥的不成问题 |
| 广告请求层 | 负责请求广告数据，其中定义了参数 builder 模式，枚举以及策略的传入 |
| 广告数据处理层 | 提取 track 数据保存到队列，解析数据生成原始数据，支持自定义转换数据 |
| 广告数据 track 层 | 保存已经上报的数据包括 id 和 上报类型，判断上报是否满足条件例如 1s 才算曝光，使用队列上报，这里使用前面的 filter 一下，已经上报就不在上报 |

1

| 角色 | 角色功能 |
|---|---|
| ADSDKEntry | 广告 SDK 切入点 |
| ADExecutor | 广告拉取执行器 |
| ADTracker | 广告追踪器 |
| ADDataStorage | 已经上报数据的存储，内存 |
| ADTrackerReporter | 具体上报类 |
| ADTrackerFilter | 上报拦截 |
| ADDataProcessor | 广告数据处理器 |
| ADRequestType | 请求类型枚举 |
| ADRetryStrategy | 请求重试策略 |

![](https://raw.githubusercontent.com/XPJ1993/images/master/guanggaoSDK.png)

