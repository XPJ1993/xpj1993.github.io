---
layout: post
title: Coroutines JobSupport
subtitle: 目前比较重要的 Coroutines 角色
image: /img/lifedoc/jijiji.jpg
tags: [Kotlin]
---

JobSupport 看名字就像 Java 里面的 LockSupport，同样地，在 Java 里面这个 LockSupport 是核心，现在 Kotlin 则是 JobSupport ，只是这里的明显看到说是要废弃这个 Api 也不知道后面怎么准备，只能拭目以待了。

| 关键词们 | 用法解析 |
|---|---|
| _state | 表明当前 Job 状态 |
| isActive | 是否是激活的状态 |
| parentHandle | 父句柄 |
| start | 开始工作 |
| completionCause | 异常发生原因 |

#### Support 核心
主要是就是可以看到状态的驱动，根据不同的状态生命周期去完成事件驱动。
这里实现了很多 Job 定义的方法，例如 Join / Cancel / await 等等， Join 内部会首先调用 joinInternal 去判断状态，如果这个没有返回想要的状态，那么会继续陷入 joinSuspend 中， Cancel 则调用的 cancelInternal 去进行 Cancel ，内部继续调用到 cancelImpl 中， 然后根据状态去判定调用 makeCancelling 还是 cancelMakeCompleting ， afterCompletion 在这些方法里面有的是通知别的角色当前角色状态的改变，有的是 进行收尾。
Start 方法会调用到 startInternal 然后 _state 会通过 cas 去置换状态，置换成功之后会调用 onStartInternal 通知这个启动成功。
parentCancelled 也会调用到自己的 cancel 这个就是协程取消传递的原因，理论上就是递归终止，由 parent 触发，但是子协程会逐级取消自己的执行。

```kotlin
// 子协程的 Cancel 则会区分情况，如果是正常 Cancel 也就是通过 CancellationException 异常去处理的，那么就不理，否则就会取消自己并且通报是否有异常发生
    public open fun childCancelled(cause: Throwable): Boolean {
        if (cause is CancellationException) return true
        return cancelImpl(cause) && handlesException
    }

// 协程核心，这里调度告知最后结果
    public final override fun invokeOnCompletion(
        onCancelling: Boolean,
        invokeImmediately: Boolean,
        handler: CompletionHandler
    ): DisposableHandle {
        var nodeCache: JobNode<*>? = null
        loopOnState { state ->
            when (state) {
                is Empty -> { // EMPTY_X state -- no completion handlers
                    if (state.isActive) {
                        // try move to SINGLE state
                        val node = nodeCache ?: makeNode(handler, onCancelling).also { nodeCache = it }
                        if (_state.compareAndSet(state, node)) return node
                    } else
                        promoteEmptyToNodeList(state) // that way we can add listener for non-active coroutine
                }
                is Incomplete -> {
                    val list = state.list
                    if (list == null) { // SINGLE/SINGLE+
                        promoteSingleToNodeList(state as JobNode<*>)
                    } else {
                        var rootCause: Throwable? = null
                        var handle: DisposableHandle = NonDisposableHandle
                        if (onCancelling && state is Finishing) {
                            synchronized(state) {
                                // check if we are installing cancellation handler on job that is being cancelled
                                rootCause = state.rootCause // != null if cancelling job
                                // We add node to the list in two cases --- either the job is not being cancelled
                                // or we are adding a child to a coroutine that is not completing yet
                                if (rootCause == null || handler.isHandlerOf<ChildHandleNode>() && !state.isCompleting) {
                                    // Note: add node the list while holding lock on state (make sure it cannot change)
                                    val node = nodeCache ?: makeNode(handler, onCancelling).also { nodeCache = it }
                                    if (!addLastAtomic(state, list, node)) return@loopOnState // retry
                                    // just return node if we don't have to invoke handler (not cancelling yet)
                                    if (rootCause == null) return node
                                    // otherwise handler is invoked immediately out of the synchronized section & handle returned
                                    handle = node
                                }
                            }
                        }
                        if (rootCause != null) {
                            // Note: attachChild uses invokeImmediately, so it gets invoked when adding to cancelled job
                            if (invokeImmediately) handler.invokeIt(rootCause)
                            return handle
                        } else {
                            val node = nodeCache ?: makeNode(handler, onCancelling).also { nodeCache = it }
                            if (addLastAtomic(state, list, node)) return node
                        }
                    }
                }
                else -> { // is complete
                    // :KLUDGE: We have to invoke a handler in platform-specific way via `invokeIt` extension,
                    // because we play type tricks on Kotlin/JS and handler is not necessarily a function there
                    if (invokeImmediately) handler.invokeIt((state as? CompletedExceptionally)?.cause)
                    return NonDisposableHandle
                }
            }
        }
    }    

// await() 实现
    internal suspend fun awaitInternal(): Any? {
        // fast-path -- check state (avoid extra object creation)
        while (true) { // lock-free loop on state
            val state = this.state
            if (state !is Incomplete) {
                // already complete -- just return result
                if (state is CompletedExceptionally) { // Slow path to recover stacktrace
                    recoverAndThrow(state.cause)
                }
                return state.unboxState()

            }
            if (startInternal(state) >= 0) break // break unless needs to retry
        }
        return awaitSuspend() // slow-path
    }

// 这里是核心的定义
public interface Continuation<in T> {
    /**
     * The context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing a successful or failed [result] as the
     * return value of the last suspension point.
     */
    // 这里封装了回调的结果。 
    public fun resumeWith(result: Result<T>)
}    

// delay 的定义
    public suspend fun delay(time: Long) {
        if (time <= 0) return // don't delay
        // 调用了 suspendCancellableCoroutine 传入了 scheduleResumeAfterDelay
        return suspendCancellableCoroutine { scheduleResumeAfterDelay(time, it) }
    }

// 使用了 Java 底层的能力。
    @InlineOnly
internal inline fun unpark(thread: Thread) {
    timeSource?.unpark(thread) ?: LockSupport.unpark(thread)
}
```

