---
layout: post
title: 观察者模式
subtitle: 一个对象的改变将导致其他一个或多个对象也发生改变，而不知道具体有多少对象将发生改变，可以降低对象之间的耦合度
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---

| 使用场景 | 实现方式 |
|---|---|
| 使用场景为一群观察者对一个被关注对象感兴趣，被观察对象将观察者添加到自己的 list 中，自身有变化了就会通知我有变化了，然后观察者通过对应方法去取自己感兴趣的 | 三个角色，一个被观察者，一些观察者，还有一个负责组合他们，值得注意的是观察者是依托在被观察者身上的，观察者模式相对于监听者不知道哪里变化了，只会通知 update |
| **注意事项** | 1. 注意内存泄漏  2. 注意线程间关系 3.注意线程同步的问题，因为可能会同时操作一个 list 4. 注意循环依赖或者观察|

**架构图**

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210901202109.png)

```java
package com.example.xpj.observer;

import java.util.ArrayList;
import java.util.List;
 
// 被观察者
public class DPSubject {
   
   private List<DPObserverable> observers 
      = new ArrayList<DPObserverable>();
   private int state;
 
   public int getState() {
      return state;
   }
 
   public void setState(int state) {
      this.state = state;
      notifyAllObservers();
   }
 
   public void attach(DPObserverable observer){
      observers.add(observer);      
   }

   public void deattach(DPAbstractObserver observer) {
       observers.remove(observer);
   }
 
   public void notifyAllObservers(){
      for (DPObserverable observer : observers) {
         observer.update();
      }
   }  
}

package com.example.xpj.observer;

// 组合他们
public class DPObserverTest {
    private DPSubject sub = new DPSubject();
    public void testObserver() {
        new DPObserver1(sub);  
        new DPObserver2(sub);
        
        System.out.println("First state");
        sub.setState(233);
        System.out.println("Second state");
        sub.setState(566);
    }
   
}

package com.example.xpj.observer;
// 观察者
public class DPObserver1 extends DPAbstractObserver{
    public DPObserver1(DPSubject subject) {
        this.sub = subject;
        sub.attach(this);
    }

    @Override
    public void update() {
        System.out.println("this : " + getClass().getName() + 
        " receive update : " + sub.getState());
    }
}
```

#### 总结

自己会把观察者模式和监听者模式弄混，主要是因为观察者模式这种和监听者还是比较相似的，另外因为用的多的 livedata 举例，livedata 的最常用方法是 observe(lifecycle, obsever)
 他这里第一个是生命周期标识，后者则是监听者自身，但是 livedata 这里和标准观察者不同的是 livedata 在 onChange 的时候会把最新的数据吐回来，并且依赖于第一个 lifecycle 的控制不需要手动去 livedata 身上 deattach 观察者，用起来就很爽，这个就是最经常用的观察者模式实践。
livedata 的另一个优势是它的最常用的使用场景是几乎同时只有一个 lifecycle 处在活跃状态，它一般也就通知一个而已，当然他支持通知多人，只是我们不常用而已，因为在实践中 livedata 也会细分，负责的数据不一样。
