---
layout: post
title: First post!
image: /img/lifedoc/jijiji.jpg
tags: [标签]
---

### 0807 组件化学习

根据别人的源码学习组件化。

组件化原理：

就是通过AAR引入不同的组件，组件又可以根据自己的不同优先级去初始化，这里主要的点就是每个组件不仅自己可以直接使用，也可以作为AAR被整体项目使用，这个就和我在360的时候是一样的，只是那时候没有使用ARouter去通信而是自己搞了个Activity代理而已。

架构图：
![组件化架构图](https://raw.githubusercontent.com/Pjex/images/master/compnent.png)

通过架构图可以看到组件化分了好多层，这里有存在的问题是同级别的互相依赖问题，因为虽然应该避免同级别互相引用，但是这个不能避免。
有几个解决方案：
一个是每个组件都有两部分组成，所有依赖的都是export以及本体，大家都应用export的那部分，他去实际调用方法；
另一个就是在底层增加api接口组件，所有的组件在这里去声明自己对外开放的能力，这个有个弊端就是大家都在改容易有冲突和遗漏；

另一个问题是生命周期的管理，初始化老哥使用的是通过apt然后根据优先级去反射调用，这里有优化的点就是可以异步，可以插桩。

还有就是需要资源加上对应的前缀，避免资源混乱。

数据通信另一个是使用eventbus定义一个基础的bean，里面封装的是根据key value的方式，自己按照自己的key获取value然后使用json语义化。

gradle统一管理，可以使用gradle去限制统一的依赖版本，例如androidx等等，否则容易出现不同的组件应用不同版本导致版本混乱。


### AQS锁家族

![锁结构](https://raw.githubusercontent.com/Pjex/images/master/20210807190956.png)


