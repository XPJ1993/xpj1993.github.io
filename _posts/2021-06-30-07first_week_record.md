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


