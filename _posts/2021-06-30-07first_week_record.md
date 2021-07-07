---
layout: post
title: 202107第一周
image: /img/lifedoc/jijiji.jpg
tags: [周报]
---

### 0630 第一天

今天完成了ASM插入任意参数的学习，主要的用法如以下code：

```kotlin
index = newLocal(Type.LONG_TYPE) // 分配一个long类型的变量，并且返回分配的索引，这个方法我看源码是因为这里继承了methodvisitor在visitframe的时候存起来的变量，如果需要改的话，就在上面改了，里面使用的是System.arrayCopy
mv.visitVarInsn(Opcodes.LSTORE, index) // 将上一个返回的值保存到这个index 后面取得时候就去lstore然后index+1去保存别的就可以了
```

咱们学东西，要有自己节奏和原则，会用不代表学会，不像某些人知道个ASM就想着乱用，实际上知道了ASM是基于对字节码的破解然后理解有哪些常用的字节码命令会更加用的得心应手，另外，Java的许多东西，asm都有vistor，学会ASM需要深刻理解visitor模式，然后需要知道字节码的常用字节码命令，然后就是看asm的源码，知道人家这样设计的好处。asm不仅仅访问method（**MethodVisitor**）他还访问很多如字段（**FieldVisitor**），注解（**AnnotationVisitor**），泛型（**SignatureVisitor**），类（**ClassVisitor**）等等。

### 0701-0702

这两天主要是处理一些杂事，完成了卡片入口和小岛入口的添加以及埋点工作。2号设计了一下广告SDK主要分哪些，做了一个粗略排期，具体模块大致如下：

![](https://raw.githubusercontent.com/Pjex/images/master/20210703135518.png)

说一些做SDK或者做任何稍微基础点的模块的原则，之前在前司做吴侃就是这样，让用的人有最少的心理负担，做到好用易用，外面接口一致简单易用，而SDK内部别有洞天，做到架构合理，稳定易扩展，各个模块与角色功能单一且强大。

嗨，说到这里，实际上虽然在前司遭受的蹂躏着实不少，但是一些设计原则了，基础功底了，都有一定的进步，目前在本司主要是有了合适的舞台去展现这些能力，实际上我的潜力无限，确实也配得上高潜这个评价啊哈哈。

下一步就是在职复习，下一步准备找不写UI的Android工作，例如虚拟化、音视频相关、架构相关、小程序容器等，目前的计划是明年三四月份之后跑路，希望在此之前不要被裁掉吧，不然可能没有足够的时间去复习了，如果在此之前没有裁掉我，那么我休陪产假的时间可以好好学习学习，看看这老二是不是我的福星了。

### 0705 学习APT

粗略看了别人写的APT文章，真不错，主要涉及了反射去获取生成的文件，通过对应的注解做功能增强等等。下面是从编译阶段拿到对应所有需要代理生成view的方式，摘抄自人家的评论。

```java
之前有位老哥给我提了个问题：通过 Android 插件获取所有 Xml 布局中控件 View 的思路？
这是个好问题啊，不知为啥又把评论给删除了😂 ，不懂就问挺好的，这里我做一下回答：
一、首先我们要对 Android 打包编译流程有一定的了解，其中 mergeDebugResources 这个 task 会对所有的 Xml 进行合并，并输出到：
1、如果是 debug 包：
/build/intermediates/incremental/mergeDebugResources/merger.xml 这个文件下
2、如果是 release 包：
/build/intermediates/incremental/mergeReleaseResources/merger.xml 这个文件下
二、自定义插件，将自己编写的 task 挂载到 mergeDebugResources 后面，然后遍历 merger.xml 这个文件就可以拿到所有的控件了
```

APT整体来说就是使用注解和对应的annotation处理器去解析对应的注解然后生成自己的辅助类等等，例如butterknife，room，retrofit等等几乎都用了APT去进行功能增强，我要学习的hilt也是基于注解进行的功能增强，注解真的是一个利器呀。

### 0706-0707 修改卡片和小岛问题

0706 主要是开各种会，然后画了SDK的架构图和部分类图。

0707 修改卡片和小岛问题，完善SDK内部类图，增加了Data收集模块。

![](https://raw.githubusercontent.com/Pjex/images/master/广告SDK架构.jpg)

抄腾讯作业，发现他们的SDK暴露的方法都是同一个出口，是使用的静态方法去暴露的，对外没有暴露内部的任何具体干活的类，但是我这边的区别是数据结构需要我定义，这部分是需要暴露的，然后传入的参数也要暴露，但是也仅仅是这些，别的应该设置为包访问权限，同样的我看腾讯的bugly也是有对应的策略啥的再暴露。

设计SDK或者任何模块都要遵循设计原则，主要是单一职责，迪米特最少知道原则，内部则应该符合依赖倒置，面向接口编程，以及接口隔离，开闭原则等，里斯替换原则子可以替爹，爹不可替换子，子都是有扩展的。

