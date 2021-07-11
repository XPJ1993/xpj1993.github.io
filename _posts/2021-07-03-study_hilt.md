---
layout: post
title: 学习hilt，做出hilt demo来
image: /img/lifedoc/jijiji.jpg
tags: [hilt]
---

继续学习，本节主要学习hilt使用，这个框架是谷歌提供的更加适合android的依赖注入框架，和这个类似的还有dagger，这个本身是给Java用的，谷歌觉得不错就搞了一个dagger2去做，目前看这个对于用户来说还是偏复杂，就跟我之前说的SDK原则一致，要把用户服务好，这个dagger2门槛那么高，明显服务的不够好吗，用户那里有那么聪明呀，我就是这些不够聪明人之一。但是hilt就不一样了，不仅仅是为Android量身定做的，同时又结合kotlin 协程等，用起来还是爽的，生命周期啥的默认的activity viewmode等等都有适配，用起来应该简单且优雅。

用完之后探究他的原理，如果我做我咋做，有没有借鉴他的思想自己做过一些类似的框架，或者优化出来呢。

**就像之前考虑过为什么他们做App还要看系统源码呢，后来看插件化，各种hook系统服务接口，或者代理，从哪里才明白技术人不应该给自己设置边界，同样滴，现在学的asm等，不也是因为人家对Java的熟悉，搞出来对字节码的各种骚操作，才使得我们可以在编译阶段去生成对应的代码而不是运行时才去确定逻辑提升效率么。**

就像之前用的butterknife黄油刀，也是使用的apt结合注解去搞得帮你findviewbyid，帮你做一个click等等，这个效率高也是因为在编译期间给搞好了对应的_binding类。我们用过这个，用过greendao，用了room会发现注解好厉害哟，可是我们自己用过那些呢，hhh，我的游客模式用过啊，到时候也可以扯，v2游客我也用了，这个注解就是一个标志，是否对应的setOnclick方法，hhh。

如果问我怎么debug，不好意思，我的remote不咋好使，我是打log的啊。哈哈哈。

### hilt思想

hilt和dagger以及dagger2一样都是依赖倒置的思想，在这个思想下他们通过不同的实现方式去实现这一思想，例如dagger是依赖动态代理的，而dagger2和hilt都是基于注解的，用多了google的东西你会发现谷歌是如此喜欢使用注解，那么注解一定有他的独到之处，通过学习我们知道注解是代码的元信息. 注解说白了就是对代码的描述，代码描述功能，注解描述代码，有了这些我们就能对代码做手术了，例如通过对声明注解的类去做 功能映射，去做扩展。

```java
Java注解又称Java标注，是Java语言5.0版本开始支持加入源代码的特殊语法元数据[1]。

Java语言中的类、方法、变量、参数和包等都可以被标注。和Javadoc不同，Java标注可以通过反射获取标注内容。在编译器生成类文件时，标注可以被嵌入到字节码中。Java虚拟机可以保留标注内容，在运行时可以获取到标注内容[2]。 当然它也支持自定义Java标注[3]。
```

习惯了注解，突然不知道dagger是怎么做的依赖倒置了，应该是接口，因为动态代理需要依赖接口去实现对应的东西，用注解的谷歌还是聪明啊，下次框架类的东西的优先考虑注解的使用。


###  0711 接入hilt

今天接入了hilt的使用，目前就是最简单的使用方式，使用几个最基本的注解去做到依赖注入的，首先就是增加了一个app作为整个app的入口点，然后在目标activity里面增加对应的入口点，这里都是增加注解的方式，例如
```kotlin
@AndroidEntryPoint // activity的注入点
@Inject // 注入的目标
@HiltAndroidApp // app注入的入口点
// 测试类，这里参数也可以注入进来，只需要在构造器前面加上注解，这里需要注意即使没有参数也要加上空的构造器方便注解声明
class GrowthTest @Inject constructor(private val inner: GrowthTestInner) 
```
实际上hilt的核心思想就是通过注解以及apt这里他们用的是kapt生成对应的辅助类，用到的时候使用factory模式去create给目标使用，这里截图一下，看看都有哪些东西

![](https://raw.githubusercontent.com/Pjex/images/master/20210711175235.png)

妈的，本来还准备深究一下hilt的核心原来，他是什么实际注解进来的，他是怎么做到懒加载的，只是看遍了代码也没有发现啊，只发现了中间产物，不明所以啊，这个不理解dagger就很难知道这玩意了，因为很多直接进去发现都是dagger internal下面的代码。

![](https://raw.githubusercontent.com/Pjex/images/master/20210711180718.png)

看一个最简单的test的生成代码：
```kotlin
public final class GrowthTest_Factory implements Factory<GrowthTest> {
  private final Provider<GrowthTestInner> innerProvider;

  public GrowthTest_Factory(Provider<GrowthTestInner> innerProvider) {
    this.innerProvider = innerProvider;
  }

  @Override
  public GrowthTest get() {// 这里可以进到dagger internal里面
    return newInstance(innerProvider.get());
  }

  // 这里的方法都不知道啥时候调用的没有追到啊
  public static GrowthTest_Factory create(Provider<GrowthTestInner> innerProvider) {
    return new GrowthTest_Factory(innerProvider);
  }
  // 这里的也是这个应该是每一个都生成的一个外部的类。
  public static GrowthTest newInstance(GrowthTestInner inner) {
    return new GrowthTest(inner);
  }
}
```

时候追究如何生成的代码，然后他的调用时机是哪个。
