---
layout: post
title: 学习计划
subtitle: 学习计划，主要是列出未来要学习的东西，然后这里更新结果，附上链接
image: /img/lifedoc/jijiji.jpg
tags: [Study]
---

### 0711 APT学习计划

学东西都是有目的的，学到这个技能可以解决什么问题，遇到类似的问题是否可以更加快速的解决等，APT早有耳闻，因为最早的butterknife就是基于APT去生成的XXXActivity_Binding.java然后编译到对应的dex中，这样就少了findView过程，也可以少了onClick等注解等等，但是因为现在谷歌的不断升级导致这玩意已经不怎么靠谱了，因为不仅仅有R文件他还有R2文件，这就有可能导致找不到对应的资源问题产生了。

除了这个还有很多优秀的库是用了APT，例如Arouter，room，greenDao，hilt，retrofit，gson等等等等，并且，很多牛逼的三方库都很偏爱使用注解去解决问题，他们喜欢对代码进行操作，注解加上反射可以解决很多的问题，然后注解配合asm也是一把利器，通过这些可以写出好用的架子。

实际的工作中也可以使用APT，ASM，反射，动态代理等这些武器去攻克一个个问题，结合自己对知识的理解，去解决项目中的痛点问题，对于编译原理的理解，哪怕到时候是自己转行做别的，有了这些坚实的基础也不怕了呀。

对于ASM，前段时间已经有了入门的基础了[ASM结合Gradle](https://pjex.github.io/2021-06-27-%E6%9D%A5%E4%B8%80%E7%AF%87asm%E7%9A%84%E7%BB%93%E5%90%88plugin/)，而对于APT虽早有耳闻但是却没有进行过实操，因为近期是对hilt的使用，在做完这个之后立马全身心投入到APT大业中。

学习了这些工程上的利器，到时候吹逼就更有自信了呢。啊哈哈。

### 0718 update
apt已经做了一个小demo了，就是类似butterknife一样做一个findViewById的功能。

### 代码规则
因为代码都在一个工程中，那么咱们怎么管理，不同的demo又不想搞一堆项目，咱不是会用git吗，那就多建对应主题的分支不就得了么。如果有两个分支的交叉那么打patch并且搞出一个混合分支，不要污染彼此独特的分支。


[APT学习一](https://pjex.github.io/2021-07-18-study-apt/)

