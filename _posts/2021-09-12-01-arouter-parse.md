---
layout: post
title: Arouter 解析
subtitle: 组件化基础之 Arouter 解析
image: /img/lifedoc/jijiji.jpg
tags: [三方库]
---

### Arouter 三方库解析

| 文章目录 | 代码解析 |
|---|---|
| 使用方式解析 | 通过在 build.gradle 以及 文件中具体使用解析怎么使用 arouter 去打开对应 path 的 activity |
| content1 | content2 |
| content1 | content2 |
| content1 | content2 |
| content1 | content2 |

#### 使用方式

build.gradle 文件
```kotlin
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'com.xpj.pbversion.versiongplugin'
    id 'kotlin-kapt'
    id 'dagger.hilt.android.plugin'
    id 'com.alibaba.arouter' // 导入对应的 plugin 
}

// default config 中添加对应的 compile 配置
javaCompileOptions {
            annotationProcessorOptions {
                includeCompileClasspath true
                // <span style="color:#871F78;">特别注意这里是 += 而不是直接等于</span>，因为我这个项目还有 hilt 以及 room 相关的 注解处理器 所以这里需要 arguments += 否则运行崩溃，别的注解解析失败
                arguments += [AROUTER_MODULE_NAME: project.getName()]
            }
        }

// 增加对应的库的引用和注解引用
    api OtherLibraryKt.arouterApi
    kapt OtherLibraryKt.arouterCompile
    annotationProcessor OtherLibraryKt.arouterCompile        
```

```kotlin
// 目标 Activity 里面增加 route 注解， <span style="color:#871F78;"> 注意这里的路径至少两层，且以 / 分割</span>
@Route(path = "/wulala/la")
class PBArouterTest : FragmentActivity() {


// 调用方使用 Arouter 调用
ARouter.getInstance().build("/wulala/la").navigation()
```

通过对上面的代码解析，可以看到最简单的用法就是在 build 文件中增加 arouter 的 api 引用，增加 apt 引用，以及在 default config 中添加对应的配置，具体这个配置好像 hilt 和 room 理论上也应该有，但是 谷歌 可能已经默默做了，因此不需要我们去给她指定 module name。

接着在 代码中使用 router 注解去声明该类对应的 path 是哪个，然后在调用的地方去通过获取 arouter 的 instance build 对应的 postcard 然后 navigation 就可以了。

#### 原理解析

经过上一节简单使用可以发现 arouter 用起来并不是很复杂，那么它的实现原理是什么呢，如果是我们自己写要怎么写从哪里入手去做一个 router 框架。

首先一个 router 框架需要一个调度中心，在别人调用的时候可以找到对应的目标并且可以启动目标去完成功能， router 本意不就是寻址和导航吗。

要想实现这个框架，那么我们可以想一下都需要哪些步骤，


| 框架步骤 | 实现方式 |
|---|---|
| 目标对象收集 | 1. 使用反射获取实现某个特定接口的类，然后收集到对应的 map 中，这个接口可以提供一个接口就是返回对应的唯一 name，调用的时候根据这个名字去寻址； 2. 善用注解，使用生命周期为 RetentionPolicy.CLASS （这种注解信息会保存在类文件中，但是运行时会去掉这个注解）的注解去做，注解中定义他的 value 为唯一区别值，使用 apt 技术去解析对应的被注解的类 |
| 目标对象存储 | 可以使用 map 去存起来目标对象，这里 key 是 String 唯一值，value 为对应的 class |
| 目标对象使用 | 因为 router 需要不仅仅跳转到 activity 也需要 fragment 或者 service，可以根据他的类信息去判断具体的对象，然后在做跳转 |
| 使用方式 | 做一个单例去对这些已经收集好的目标做寻址，找到就导航过去 |
| 使用参数 | 这个可以定义一个序列化的数据作为参数，通过解析这个参数赋值给 intent 然后再去打开对应目标的时候将参数给到他们 |
| content1 | content2 |

#### arouter 是怎么做

##### 目标收集

arouter


