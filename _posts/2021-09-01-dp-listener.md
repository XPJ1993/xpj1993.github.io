---
layout: post
title: 监听者模式
subtitle: 一般用于针对单一事件的超强烈兴趣
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---

| 使用场景 | 实现方式 |
|---|---|
| 使用场景为对某一事件或者数据强烈感兴趣，listener 会回调感兴趣的数据回来，Android 中使用最多的是 setOnClickListener | 有一个被监听者，他负责收集监听者并且在有事件或者数据产生的时候遍历回调给监听者，注意 register 与 unregister 要成对出现 |
| **注意事项** | 1. 注意内存泄漏  2. 注意线程间关系 3.注意线程同步的问题，因为可能会同时操作一个 list |

```kotlin
package com.xpj.studypattern.listener

import com.xpj.designpartten.PRTMsg
import java.util.*

/**
 * author : xpj
 * date : 9/1/21 11:10 AM
 * description :
 */
class ListenerTest {
    private val sender = ListenerSender()

    fun test() {
        sender.addListener(object : IListener {
            override fun notifyNewValue(value: Int) {
                PRTMsg("receive value is : $value")
            }
        })

        sender.testNotify()
    }
}

private class ListenerSender {
    private val listeners = LinkedList<IListener>()

    fun addListener(listener: IListener) {
        synchronized(this) {
            listeners.add(listener)
        }
    }

    fun removeListener(listener: IListener) {
        synchronized(this) {
            listeners.remove(listener)
        }
    }

    fun testNotify() {
        synchronized(this) {
            listeners.forEach {
                it.notifyNewValue(233)
            }
        }
    }
}
```
