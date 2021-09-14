---
layout: post
title: Kotlin 特性学习（2）协程学习
subtitle: Kotlin 协程学习
image: /img/lifedoc/jijiji.jpg
tags: [Kotlin]
---

### 协程常用词

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210824200923.png)

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210824201000.png)

### 协程特性

![](https://raw.githubusercontent.com/XPJ1993/images/master/20210824200804.png)

```kotlin

// 几个作用域中的问题
    private fun testCoroutineScope1() = runBlocking {
        println("testCoroutineScope")
        launch {
            delay(800L)
            println("${System.currentTimeMillis()} launch this : $this  coroutine context: ${this.coroutineContext}")
        }

        coroutineScope {
            launch {
                delay(500)
                println("${System.currentTimeMillis()} coroutineScope inner this $this launch context: ${this.coroutineContext}")
            }

            delay(1100)
            println("${System.currentTimeMillis()} coroutineScope this $this context ${this.coroutineContext}")
        }

        println("${System.currentTimeMillis()} runBlocking this $this context ${this.coroutineContext}")
    }

// 输出结果，根据delay的时间不同，先后按照delay之后执行的
testCoroutineScope
// coroutineScope inner 里面的 context 是 StandaloneCoroutine
1629689386348 coroutineScope inner this StandaloneCoroutine{Active}@4f8e5cde launch context: [StandaloneCoroutine{Active}@4f8e5cde, BlockingEventLoop@759ebb3d]
// 最外层 launch StandaloneCoroutine
1629689386648 launch this : StandaloneCoroutine{Active}@4cdf35a9  coroutine context: [StandaloneCoroutine{Active}@4cdf35a9, BlockingEventLoop@759ebb3d]
// coroutineScope 使用的上下文作用域为 ScopeCoroutine，与 runBlocking 功能类似，等待内部执行完毕才会关闭，但是他会挂起，而不是阻塞
1629689386945 coroutineScope this ScopeCoroutine{Active}@4c98385c context [ScopeCoroutine{Active}@4c98385c, BlockingEventLoop@759ebb3d]
// runBlocking 没有使用 launch 的上下文为 BlockingCoroutine，默认是主线程的且会阻塞主线程
1629689386946 runBlocking this BlockingCoroutine{Active}@5fcfe4b2 context [BlockingCoroutine{Active}@5fcfe4b2, BlockingEventLoop@759ebb3d]


```

**suspend 用法示例**

```kotlin

    suspend fun doWorld() {
        delay(500)
        println("World!")
    }

// 这里如果不加 launch 就会直接输出，实际上 doWorld 和普通函数不同就是加上 suspend 关键字，但是没有 launch 就不会产生 Job 也就无法产生 delay 挂起的效果。
    fun doHello() = runBlocking {
        launch {
            doWorld()
        }
        println("hello")
    }


// launch 定义，可以看到这里最终是生成了一个 Job ，这个 Job 是一个协程
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```
