---
layout: post
title: 状态机模式
subtitle: 一般用于有强烈顺序的程序，例如生命周期等
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---

话不多说直接上代码，状态机模式主要是用在有生命周期或者每个阶段有依赖的情况下，例如视频编解码就会有强烈的生命周期如果跨周期或者没有准备好等等情况就会抛出 IllegalStateException 这个我在做视频存储的时候算是见识到了。

另一个是在做配置工具的时候，因为配置需要是一步步来的，因此这个使用状态机模式也是极适合的，不过这个更加复杂些，因为刚开始配置一台的时候顺序配置没啥问题，后来并发配置多台的时候就需要有角色去管控对应的机器是否到了目标状态了，如果没到则不驱动到哪里，不配置那一项或者提示配置错误。

show 刚刚撸的 code。

```kotlin
package com.xpj.studypattern.statusmachine

/**
 * author : xpj
 * date : 8/31/21 8:13 PM
 * description : 拥有四个状态，状态是依次递增的不可以跨状态
 */
enum class SMStatus {
    INIT,
    START,
    RUNNING,
    STOP
}

package com.xpj.studypattern.statusmachine

import com.xpj.designpartten.PRTMsg
import java.lang.IllegalStateException

/**
 * author : xpj
 * date : 8/31/21 8:15 PM
 * description :
 */
class SMTest {
    private var cur = SMStatus.INIT

    private fun updateStatus(nStatus: SMStatus): Boolean {
        return if (nStatus == validStatus()) {
            PRTMsg("$cur to $nStatus success!!!")
            cur = nStatus
            true
        } else {
            // 如果对不上那么就抛出异常，一般这里去控制或者实际调度做什么工作，例如配置工具哪里，我就是在上一状态驱动下一状态的，当然的也有的外界可以控制走到什么状态.
            throw IllegalStateException(
                "Target status is $nStatus , cur is $cur , " +
                        "need ${validStatus()}"
            )
            false
        }
    }

    private fun initMachine() {
        PRTMsg("I want to INTI status")
        updateStatus(SMStatus.INIT)
    }

    private fun runningMachine() {
        PRTMsg("I want to running status")
        updateStatus(SMStatus.RUNNING)
    }

    private fun startMachine() {
        PRTMsg("I want to start status")
        updateStatus(SMStatus.START)
    }

    private fun stopMachine() {
        PRTMsg("I want to stop status")
        updateStatus(SMStatus.STOP)
    }

    fun testMachine() {
        startMachine()
        runningMachine()
        stopMachine()
        initMachine()
        runningMachine()
    }

    private fun validStatus(): SMStatus {
        return when (cur) {
            SMStatus.INIT -> SMStatus.START
            SMStatus.START -> SMStatus.RUNNING
            SMStatus.RUNNING -> SMStatus.STOP
            SMStatus.STOP -> SMStatus.INIT
        }
    }
}
```

### 彩蛋蛋蛋

提交代码历史，哈哈哈，看起来还是挺多的呀。

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210831201929.png)




