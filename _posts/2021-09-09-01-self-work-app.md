---
layout: post
title: 对讲 App 架构
subtitle: 搞一个挣钱的对讲 App
image: /img/lifedoc/jijiji.jpg
tags: [Kotlin]
---

使用到的技术以及架构，不断完善，现在仅仅是抛砖引玉，引以后自己的玉

| 使用到的技术 | 具体实现 |
|---|---|
| v3 版本的混淆 | 混淆 proguade 文件自己保存好，使用 v3 加密 |
| 多版本打包 | 使用美团 walle 方案 |
| 整体使用模块化开发 | 不同功能主题不同 module |
| 网络使用 square 全家桶 | retrofit 加 okhttp3 |
| 图片缓存框架使用 glide | 使用 glide 做图片加载 |
| 数据库用 room | 如果使用数据库那么就是用 room |
| 整体 mvvm | 基于 mvvm 开发 |
| 使用 kotlin flow 开发 | 使用 flow 结合 mvvm |
| 使用 协程进行并发与异步 | 完全使用协程进行异步开发 |
| 使用 hilt | 使用 hilt 依赖注入 |
| sip 协议 | jainsip https://github.com/RestComm/restcomm-android-sdk |
| 开源音视频 sdk | 使用 linphone 进行音视频开发 |
| 使用一个专门的模块管理引入包控制 | 一个 module 去控制不同 module 的公共引用 |
| 信令相关需要封装为统一模块 | 信令模块 |
| 权限相关的使用 permissionX | guolin 的 permissionX 库去解决权限问题 |
| content1 | content2 |
