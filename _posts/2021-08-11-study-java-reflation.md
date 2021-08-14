---
layout: post
title: 复习Java（一）
subtitle: 反射与动态代理学习
image: /img/lifedoc/jijiji.jpg
tags: [Java]
---

### 反射

首先是反射相关的结构。

![反射相关类](https://raw.githubusercontent.com/Pjex/images/master/fansheclasspic.awebp)


Java 反射的主要组成部分有4个：
**Class**：任何运行在内存中的所有类都是该 Class 类的实例对象，每个 Class 类对象内部都包含了本来的所有信息。记着一句话，通过反射干任何事，先找 Class 准没错！
**Field**：描述一个类的属性，内部包含了该属性的所有信息，例如数据类型，属性名，访问修饰符······
**Constructor**：描述一个类的构造方法，内部包含了构造方法的所有信息，例如参数类型，参数名字，访问修饰符······
**Method**：描述一个类的所有方法（包括抽象方法），内部包含了该方法的所有信息，与Constructor类似，不同之处是 Method 拥有返回值类型信息，因为构造方法是没有返回值的。


反射中的用法有非常非常多，常见的功能有以下这几个：

1. 在运行时获取一个类的 Class 对象
2. 在运行时构造一个类的实例化对象
3. 在运行时获取一个类的所有信息：变量、方法、构造器、注解


#### 获取类对象

```java
// 获取Class 方式1

Class cc = JGTest1.class

// 获取Class 方式2

JGTest1 jgt1 = new JGTest1();
Class cc1 = jgt1.getClass();

// 通过反射获取Class 方式3
    private Class<?> getJGTClass() throws Exception {
        return Class.forName("com.xpj.javagrowth.javaref.JGTest1");
    }

// 反射实例化 1
Object jgt1 = c1.newInstance();

// 反射实例化 2
Constructor<?> jgt2 = c1.getDeclaredConstructor(String.class);// 首先获取有对应构造参数的构造器

// 传入对应的实例化参数进行实例化
        Object jgt2s = jgt2.newInstance("呜啦啦啦");

```

#### 反射获取对象的属性

```java

```


### 动态代理



