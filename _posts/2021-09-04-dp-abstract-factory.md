---
layout: post
title: 抽象工厂模式
subtitle: 一般用于产生某种类或者结果，使用抽象工厂可以屏蔽生产细节，方便迭代更新
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---

| 使用场景 | 实现方式 |
|---|---|
| 构造复杂对象，且对象细节不想被用户知道 | 定义简单工厂或者抽象工厂去完成对象的创建，一般使用抽象工厂好点，因为可以替换为二代工厂，或者在工厂内替换生产者的实现类 |
| 在三方库中发威 | hilt 使用对应的 factory 构造对应的类，例如 我们使用 inject 注解定义的注入项就会生成的对应 factory 例如下面代码 GrowthTest |
| **项目中实践** | 在广告 SDK 中使用 工厂模式产生对应的处理类 |

```java

package com.xpj.mygrowthpath;

import dagger.internal.Factory;
import javax.annotation.Generated;
import javax.inject.Provider;

@Generated(
    value = "dagger.internal.codegen.ComponentProcessor",
    comments = "https://dagger.dev"
)
@SuppressWarnings({
    "unchecked",
    "rawtypes"
})
public final class GrowthTest_Factory implements Factory<GrowthTest> {
  private final Provider<GrowthTestInner> innerProvider;

  public GrowthTest_Factory(Provider<GrowthTestInner> innerProvider) {
    this.innerProvider = innerProvider;
  }

  @Override
  public GrowthTest get() {
    return newInstance(innerProvider.get());
  }

  public static GrowthTest_Factory create(Provider<GrowthTestInner> innerProvider) {
    return new GrowthTest_Factory(innerProvider);
  }

  public static GrowthTest newInstance(GrowthTestInner inner) {
    return new GrowthTest(inner);
  }
}


```

**抽象工厂架构图**

![](https://raw.githubusercontent.com/XPJ1993/images/master/abstractFactory.png)


```java
package com.example.xpj.factorys;

import com.example.xpj.DPConstants;

public class DPFactory {
    public enum PhoneBrand {
        MI, IPHONE, MOTO
    }

    public enum PCBrand {
        LENOVO, MICROSOFT
    }

    public interface Phone {
        String brand();
    }

    public interface PC {
        String brand();
    }

    public static class Mi implements Phone {
        @Override
        public String brand() {
            return "mi";
        }
    }

    public static class IPhone implements Phone {
        @Override
        public String brand() {
            return "iphone";
        }
    }

    public static class MOTO implements Phone {
        @Override
        public String brand() {
            return "moto";
        }
    }

    public static class Lenovo implements PC {
        @Override
        public String brand() {
            return "lenovo";
        }
    }

    public static class Mirosoft implements PC {
        @Override
        public String brand() {
            return "mirosoft";
        }
    }

    public static class PhoneFactory {
        public Phone createPhone(PhoneBrand brand) {
            switch (brand) {
                case MI:
                    return new Mi();
                case IPHONE:
                    return new IPhone();
                default:
                    return new MOTO();
            }
        }
    }

    public void testSimpleFactory() {
        PhoneFactory factory = new PhoneFactory();
        Phone mi = factory.createPhone(PhoneBrand.MI);
        Phone moto = factory.createPhone(PhoneBrand.MOTO);
        Phone iPhone = factory.createPhone(PhoneBrand.IPHONE);
        DPConstants.PRTMsg(" mi is " + mi + " moto is: " + moto + " iPhone is: " + iPhone);
    }

    public interface InnerDPFactory {
        Phone createPhone(PhoneBrand brand);

        PC createPC(PCBrand brand);
    }

    public static class AbstractDPFactoryImpl implements InnerDPFactory {
        PhoneFactory phonef = new PhoneFactory();

        @Override
        public Phone createPhone(PhoneBrand brand) {
            return phonef.createPhone(brand);
        }

        @Override
        public PC createPC(PCBrand brand) {
            switch (brand) {
                case MICROSOFT:
                    return new Mirosoft();
                default:
                    return new Lenovo();
            }
        }
    }

    public void testAbstractFactory() {
        AbstractDPFactoryImpl factoryImpl = new AbstractDPFactoryImpl();
        DPConstants.PRTMsg("AbstractDPFactoryImpl produce phone: " + factoryImpl.createPhone(PhoneBrand.MI).brand()
                + " pc is : " + factoryImpl.createPC(PCBrand.LENOVO).brand());
    }
}
```

#### 总结

工厂模式在框架内使用时很好用的，factory 模板代码可以产生意想不到的效果，啥效果，就是可以非常自然的去使用目标产物而不用关心构造的细节，并且还有一个很神奇的功效就是 **只要接口稳定在工厂里面换具体实现类是很自然的事情，这样就能达到外部根本无感知，但是通过工厂我们达到了偷梁换柱的升级效果，代码越写越好，由于这个模式的存在用户基本上就是无痛插入，太强了。**
