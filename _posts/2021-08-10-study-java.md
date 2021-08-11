---
layout: post
title: 复习Java（一）
subtitle: 泛型学习
image: /img/lifedoc/jijiji.jpg
tags: [Java]
---

## Java泛型学习

### 泛型机制

![](https://raw.githubusercontent.com/Pjex/images/master/ef3e9a525a5e44d29c5785ee811d1f90_tplv-k3u1fbpfcp-zoom-crop-mark_1304_1304_1304_734.awebp)


### 泛型优点

从上面的两个例子我们可以直观的得出泛型机制的优点
1.使用泛型可以编写模板代码来适应任意类型，减少重复代码
2.使用时不必对类型进行强制转换,方便且减少出错机会

### 泛型擦除

大家都知道，Java的泛型是伪泛型，这是因为Java在编译期间，所有的泛型信息都会被擦掉，正确理解泛型概念的首要前提是理解类型擦除。
Java的泛型基本上都是在编译器这个层次上实现的，在生成的字节码中是不包含泛型中的类型信息的，使用泛型的时候加上类型参数，在编译器编译的时候会去掉，这个过程就是泛型擦除。
原因是泛型是Java 1.5才有的特性，为了兼容1.5 版本之前的代码做的妥协。虽然擦除了，但是擦除之前具体的信息 Java 但是某些（**声明侧**的泛型，接下来解释） 泛型信息会被class文件 以Signature的形式 保留在Class文件的Constant pool中。

例如在Method或者Filed或Class都有对应的 getSignature 相关方法。

#### retrofit是怎样使用泛型的

可以看出,retrofit是通过getGenericReturnType来获取类型信息的jdk的Class 、Method 、Field 类提供了一系列获取 泛型类型的相关方法。以**Method**为例,getGenericReturnType获取带泛型信息的返回类型 、 getGenericParameterTypes获取带泛型信息的参数类型。

#### Gson怎样使用泛型

所以Gson使用了一个巧妙的方法来获取泛型类型：
1.创建一个泛型抽象类TypeToken <T> ，这个抽象类不存在抽象方法，因为匿名内部类必须继承自抽象类或者接口。所以才定义为抽象类。
2.创建一个 继承自TypeToken的匿名内部类， 并实例化泛型参数TypeToken<String>
3.通过**class类**的public Type getGenericSuperclass()方法，获取带泛型信息的父类Type，也就是TypeToken<String>

**总结：Gson利用子类会保存父类class的泛型参数信息的特点。 通过匿名内部类实现了泛型参数的传递。**

#### 声明侧与使用侧

**声明侧泛型**主要指以下内容
1.泛型类，或泛型接口的声明 
```java
// 属于1
public interface IIddd<T> {
    public void test(T t)
}

// 属于2
private T getT() {}

// 属于3
public class CCddd<T> {
    T t;
}
```
2.带有泛型参数的方法
3.带有泛型参数的成员变量

**使用侧泛型**
也就是方法的局部变量,方法调用时传入的变量。

### PECS 原则，kotlin 的 in out和这个类似
PECS是从集合的角度出发的
1.如果你只是从集合中取数据，那么它是个生产者，你应该用extend
2.如果你只是往集合中加数据，那么它是个消费者，你应该用super
3.如果你往集合中既存又取，那么你不应该用extend或者super

#### 举例子
在List<? extends Fruit>的泛型集合中，对于元素的类型，编译器只能知道元素是继承自Fruit，具体是Fruit的哪个子类是无法知道的。 所以「向一个无法知道具体类型的泛型集合中插入元素是不能通过编译的」。但是由于知道元素是继承自Fruit，所以从这个泛型集合中取Fruit类型的元素是可以的。
在List<? super Apple>的泛型集合中，元素的类型是Apple的父类，但无法知道是哪个具体的父类，因此「读取元素时无法确定以哪个父类进行读取」。 插入元素时可以插入Apple与Apple的子类，因为这个集合中的元素都是Apple的父类，子类型是可以赋值给父类型的。


原文出处：https://juejin.cn/post/6978833860284907527

