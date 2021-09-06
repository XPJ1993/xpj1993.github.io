---
layout: post
title: 责任链模式
subtitle: 这种模式适合在一件事情有多个模块或者类可以处理，然后处理完事情之后的可以决定改事情是否可以继续被处理
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---

| 使用场景 | 实现方式 |
|---|---|
| 该模式适合应用场景为对一个目标事件的处理是互斥的，也就是说这个如果某一事件有人处理了，需要告诉下游处理者改事件已经被消费，下游根据这个指示就不做多余处理 | 正常情况下是事件的消费者之间是有关联的，也就是说假如有 a b 事件处理者，例如 a 持有 b， 那么可以在 a 中首先掉一共 b 消费或者处理改事件的方法，如果该方法标明自己已经处理，那么 a 就不会继续改事件 |
| **项目中实践** | 做游客模式的时候对某个事件有多个处理者，一个处理了之后就不会去继续流转该事件 |
| 我在项目中的实现方式 | 还有一种实现方式是 这些事件处理者有一个统一的处理器，例如我在游客模式那样，只要判断有处理器反应处理，那么事件就不会继续流转，这种方式就是增加了一个协调处理器如何消费事件的角色 |

**架构图**

![](https://raw.githubusercontent.com/XPJ1993/images/master/responsbilityPattern.png)

项目中的用法，这个更像是第二个了。

```kotlin
package cn.xckj.talk.ui.moments.parentcontrol.interfaces

import androidx.fragment.app.FragmentActivity
import cn.xckj.talk.ui.moments.parentcontrol.beans.PCStatusBean

abstract class PCAbstractProcessor {
    private var next: PCAbstractProcessor? = null

    fun setNextProcessor(next: PCAbstractProcessor?) {
        this.next = next
    }

    internal fun process(activity: FragmentActivity?, bean: PCStatusBean) {
        if (!doProcess(activity, bean)) {
            next?.apply {
                process(activity, bean)
            }
        }
    }

    open fun lateProcess() {

    }

    open fun cancelLateProcess() {

    }

    abstract fun doProcess(activity: FragmentActivity?, bean: PCStatusBean) : Boolean
}
```

使用集中管理式的代码样例。

```java
package com.example.xpj.responsibility;

import java.util.LinkedList;
import java.util.List;

import com.example.xpj.DPConstants;

public class DPResponsibility {
    public static class ResEvent {
        int id = 666;
    }

    public static abstract class ResProcessor {
        public abstract boolean process(ResEvent event);
    }

    public static class ProcessorController {
        private final List<ResProcessor> processors = new LinkedList<>();

        public void addProcessor(ResProcessor processor) {
            synchronized (processors) {
                // 实际上这里需要去重
                processors.add(processor);
            }
        }

        // remove processor 省略

        public void processEvent(ResEvent event) {
            for (ResProcessor resProcessor : processors) {
                if (resProcessor.process(event)) {
                    DPConstants.PRTMsg("this event: " + event + " have process by: " + resProcessor + " processors is : " + processors);
                    break;
                }
            }
        }
    }

    public static class ResProcessorA extends ResProcessor {
        @Override
        public boolean process(ResEvent event) {
            if (event.id == 66) {
                event.id = 233;
                DPConstants.PRTMsg("yes baby. i use this event. " + this);
                return true;
            }
            return false;
        }
    }

    public static class ResProcessorB extends ResProcessor {
        @Override
        public boolean process(ResEvent event) {
            if (event.id == 666) {
                event.id = 2336;
                DPConstants.PRTMsg("yes baby. i use this event. " + this);
                return true;
            }
            return false;
        }
    }

    public void testResponsibility() {
        DPConstants.PRTMsg("IM in DPResponsiblity.");
        ResEvent event = new ResEvent();
        event.id = 66;
        ProcessorController controller = new ProcessorController();
        controller.addProcessor(new ResProcessorA());
        controller.addProcessor(new ResProcessorB());
        controller.processEvent(event);
    }
}
```

#### 总结

责任链模式用武之地为 view 的事件传递过程中，有些事件是父 view 优先处理，有些视图是子 view 优先处理，通过这种责任链的形式去精确控制着事件给谁去消费。
我们的实际项目中也可以有这种形式去处理事件，但是前提是这些处理器拥有互斥的特性，否则这个模式则不匹配。
