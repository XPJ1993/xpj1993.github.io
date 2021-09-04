---
layout: post
title: 抽象工厂模式
subtitle: 一般用于产生某种类或者结果，使用抽象工厂可以屏蔽生产细节，方便迭代更新
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---

| 使用场景 | 实现方式 |
|---|---|
| 某些模块的输出不能直接匹配使用模块的格式，需要在中间增加一层 adapter 将输出转换为用户模块感兴趣的输入类型 | 定义 adapter 接口，按需转换为目标类型 |
| **项目中实践** | 最多的是 recyclerview 里面数据转换为对应的 holder 的应用 |

**架构图**

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210902143631.png)


```java
package com.example.xpj.adapters;

import com.example.xpj.DPConstants;

public class DPFoodFactory {
    // 食物生产者，每种类型只能生产自己擅长的食物
    public interface DPFood<T> {
        T produceFood();
    }

    public static class DPIntFood implements DPFood<Integer> {
        @Override
        public Integer produceFood() {
            return 2333;
        }
    }

    public static class DPLongFood implements DPFood<Long> {
        @Override
        public Long produceFood() {
            return 6666L;
        }
    }

    // adapter 的 接口，有两个泛型一个输入一个输出
    public interface DPAdapter<T, R> {
        R adapt(T t);
    }

    public static class DPInt2StringAdapter implements DPAdapter<Integer, String> {
        @Override
        public String adapt(Integer t) {
            return t.toString() + " im from Int. ";
        }
    }

    public static class DPLong2StringAdapter implements DPAdapter<Long, String> {
        @Override
        public String adapt(Long t) {
            return t.toString() + " im From Long. ";
        }
    }

    // 这里用户只能吃 String 的食物
    public static class DPAdapterUser {
        public void feedMe(String food) {
            DPConstants.PRTMsg("yes this food is i like! " + food + " thank you!!");
        }
    }

    public void testAdapter() {
        DPAdapterUser user = new DPAdapterUser();
        // 调用对应的 adapter 去转换为用户能够消化的食物
        user.feedMe(new DPInt2StringAdapter().adapt(new DPIntFood().produceFood()));
        user.feedMe(new DPLong2StringAdapter().adapt(new DPLongFood().produceFood()));
    }
}
```

#### 总结

适配器模式用的最多的是 recyclerview 里面数据与 holder 的转换，自己常用的一种类似于 adapter 的是数据转换，就是在写首页的时候有一个 transform 模块，这个也是将输入转换为我们感兴趣的模块。
