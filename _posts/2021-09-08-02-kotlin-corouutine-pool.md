---
layout: post
title: kotlin 协程的底层支持之线程池
subtitle: 协程中的线程池
image: /img/lifedoc/jijiji.jpg
tags: [ThreadPool]
---

kotiln 线程池也是一个神奇的物种啊。

| kotlin 线程池用法 | 线程池原理 |
|---|---|
| kotlin 线程池的用法为在使用默认的 Dispatchers.IO 或者其他时，他们本身底层是基于线程池做的协程支持，这里的协程类似于 Java 线程池里的 Runnable 类似，都是一个个任务 | 和 Java 线程池类似但是实现不同，有主动 steal 的动作 |

### kotlin 线程池是怎么进去的

![](https://raw.githubusercontent.com/XPJ1993/images/master/kotlin协程调用图.png)

### kotlin 线程池解析

一样滴，kotlin 线程池也是 task 的历险记
 .1
 .1
##### task 进入魔鬼池
 .1
 .2
```kotlin
// 陷入了协程中的本质，这里是分发 block 
    fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, tailDispatch: Boolean = false) {
        trackTask() // this is needed for virtual time support
        // 首先包装为 task
        val task = createTask(block, taskContext)
        // try to submit the task to the local queue and act depending on the result
        // 首先是获取当下的 worker 是那个
        val currentWorker = currentWorker()
        val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)
        if (notAdded != null) {
            // 这里确认没有添加到一个具体的 worker 的队列里面就会抛出拒绝异常，因为这个 global 的 queue 也拒绝添加了，证明现在全忙且队列已满 global queue 是使用 LockFreeTaskQueue 实现的，但是因为这个场景下我们是多个消费者，因此也是有锁的
            if (!addToGlobalQueue(notAdded)) {
                // Global queue is closed in the last step of close/shutdown -- no more tasks should be accepted
                throw RejectedExecutionException("$schedulerName was terminated")
            }
        }
        val skipUnpark = tailDispatch && currentWorker != null
        // Checking 'task' instead of 'notAdded' is completely okay
        if (task.mode == TASK_NON_BLOCKING) {
            if (skipUnpark) return
            // 给 cpu 密集型工作发信号，这内部会尝试创建 worker
            signalCpuWork()
        } else {
            // 给阻塞工作滴 worker 发信号
            // Increment blocking tasks anyway
            signalBlockingWork(skipUnpark = skipUnpark)
        }
    }

    // 这个方法定义是可以为空的时候调用的如果 worker 为空那么就直接返回 task
    private fun Worker?.submitToLocalQueue(task: Task, tailDispatch: Boolean): Task? {
        // //
    }
```
.1
 
.1
##### 负责消化 task 的魔鬼 worker
.1
.1
```kotlin
    internal inner class Worker private constructor() : Thread() {
        init {
            isDaemon = true
        }

        // guarded by scheduler lock, index in workers array, 0 when not in array (terminated)
        @Volatile // volatile for push/pop operation into parkedWorkersStack
        var indexInArray = 0
            set(index) {
                name = "$schedulerName-worker-${if (index == 0) "TERMINATED" else index.toString()}"
                field = index
            }

        constructor(index: Int) : this() {
            indexInArray = index
        }

        inline val scheduler get() = this@CoroutineScheduler

        @JvmField
        val localQueue: WorkQueue = WorkQueue()

        /**
         * Worker state. **Updated only by this worker thread**.
         * By default, worker is in DORMANT state in the case when it was created, but all CPU tokens or tasks were taken.
         * Is used locally by the worker to maintain its own invariants.
         */
        @JvmField
        var state = WorkerState.DORMANT

        /**
         * Worker control state responsible for worker claiming, parking and termination.
         * List of states:
         * [PARKED] -- worker is parked and can self-terminate after a termination deadline.
         * [CLAIMED] -- worker is claimed by an external submitter.
         * [TERMINATED] -- worker is terminated and no longer usable.
         */
        val workerCtl = atomic(CLAIMED)

        /**
         * It is set to the termination deadline when started doing [park] and it reset
         * when there is a task. It servers as protection against spurious wakeups of parkNanos.
         worker 什么时候到达他生命的尽头，在创建函数哪里我们可以看到是默认最多活 60s 的
         */
        private var terminationDeadline = 0L

        /** worker 是否到堆栈中待命了 */
        @Volatile
        var nextParkedWorker: Any? = NOT_IN_STACK

        /*
         * The delay until at least one task in other worker queues will  become stealable.
         等待啥时候可以 steal 别人的 task呀，我好饿。
         */
        private var minDelayUntilStealableTaskNs = 0L

        private var rngState = Random.nextInt()

        /**
         * Tries to acquire CPU token if worker doesn't have one
         获取 cpu 时间的控制权
         * @return whether worker acquired (or already had) CPU token
         */
        private fun tryAcquireCpuPermit(): Boolean = when {
            state == WorkerState.CPU_ACQUIRED -> true
            this@CoroutineScheduler.tryAcquireCpuPermit() -> {
                state = WorkerState.CPU_ACQUIRED
                true
            }
            else -> false
        }

        /**
         * Releases CPU token if worker has any and changes state to [newState].
         * Returns `true` if CPU permit was returned to the pool
         */
        internal fun tryReleaseCpu(newState: WorkerState): Boolean {
            val previousState = state
            val hadCpu = previousState == WorkerState.CPU_ACQUIRED
            if (hadCpu) releaseCpuPermit()
            if (previousState != newState) state = newState
            return hadCpu
        }

        // 这里直接等于 runWorker 了
        override fun run() = runWorker()

        @JvmField
        var mayHaveLocalTasks = false

        // 魔鬼开动
        private fun runWorker() {
            var rescanned = false
            while (!isTerminated && state != WorkerState.TERMINATED) {
                // 找到我的食物了
                val task = findTask(mayHaveLocalTasks)
                // Task found. Execute and repeat
                if (task != null) {
                    rescanned = false
                    minDelayUntilStealableTaskNs = 0L
                    // 消化他
                    executeTask(task)
                    continue
                } else {
                    mayHaveLocalTasks = false
                }
                
                // 等待 steal 的时间不为 0，那么就会释放 cpu 并且清空 interrupt 标志，然后阻塞 minDelayUntilStealableTaskNs 这段时间
                if (minDelayUntilStealableTaskNs != 0L) {
                    if (!rescanned) {
                        rescanned = true
                    } else {
                        rescanned = false
                        // 释放 cpu
                        tryReleaseCpu(WorkerState.PARKING)
                        interrupted()
                        LockSupport.parkNanos(minDelayUntilStealableTaskNs)
                        minDelayUntilStealableTaskNs = 0L
                    }
                    continue
                }
                 // 这里是 park 自己，如果 park 期间没人唤醒，那么我们就要消亡了
                tryPark()
            }
            // 释放 cpu 资源
            tryReleaseCpu(WorkerState.TERMINATED)
        }

        // Counterpart to "tryUnpark"
        private fun tryPark() {
            // 如果不在栈中那么就把它放到栈中，等待被调用
            if (!inStack()) {
                parkedWorkersStackPush(this)
                return
            }
            assert { localQueue.size == 0 }
            // 更新状态为 PARKED
            workerCtl.value = PARKED // Update value once
            // 如果在栈中那么就去释放资源并且进入 park 状态
            while (inStack()) { // Prevent spurious wakeups
                if (isTerminated || state == WorkerState.TERMINATED) break
                tryReleaseCpu(WorkerState.PARKING)
                interrupted() // Cleanup interruptions
                park()
            }
        }

        private fun inStack(): Boolean = nextParkedWorker !== NOT_IN_STACK

        // 我要吃，我具体怎么吃，第一步 reset idle、第二步 before task 通知、第三步 run Safely、第四步 after task 收尾
        private fun executeTask(task: Task) {
            val taskMode = task.mode
            idleReset(taskMode)
            beforeTask(taskMode)
            runSafely(task)
            afterTask(taskMode)
        }

        private fun beforeTask(taskMode: Int) {
            if (taskMode == TASK_NON_BLOCKING) return
            // Always notify about new work when releasing CPU-permit to execute some blocking task
            if (tryReleaseCpu(WorkerState.BLOCKING)) {
                // 发信号可以消耗任务了
                signalCpuWork()
            }
        }

        private fun afterTask(taskMode: Int) {
            if (taskMode == TASK_NON_BLOCKING) return
            decrementBlockingTasks()
            val currentState = state
            // Shutdown sequence of blocking dispatcher
            if (currentState !== WorkerState.TERMINATED) {
                assert { currentState == WorkerState.BLOCKING } // "Expected BLOCKING state, but has $currentState"
                state = WorkerState.DORMANT
            }
        }

        /*
         * Marsaglia xorshift RNG with period 2^32-1 for work stealing purposes.
         * ThreadLocalRandom cannot be used to support Android and ThreadLocal<Random> is up to 15% slower on Ktor benchmarks
         */
        // 提高 steal 效率 
        internal fun nextInt(upperBound: Int): Int {
            var r = rngState
            r = r xor (r shl 13)
            r = r xor (r shr 17)
            r = r xor (r shl 5)
            rngState = r
            val mask = upperBound - 1
            // Fast path for power of two bound
            if (mask and upperBound == 0) {
                return r and mask
            }
            return (r and Int.MAX_VALUE) % upperBound
        }

        // park 住了，park 期间每人呼唤我那我就要 die 了啊
        private fun park() {
            // set termination deadline the first time we are here (it is reset in idleReset)
            if (terminationDeadline == 0L) terminationDeadline = System.nanoTime() + idleWorkerKeepAliveNs
            // actually park
            LockSupport.parkNanos(idleWorkerKeepAliveNs)
            // try terminate when we are idle past termination deadline
            // note that comparison is written like this to protect against potential nanoTime wraparound
            if (System.nanoTime() - terminationDeadline >= 0) {
                terminationDeadline = 0L // if attempt to terminate worker fails we'd extend deadline again
                tryTerminateWorker()
            }
        }

        /**
         * Stops execution of current thread and removes it from [createdWorkers].
         */
        private fun tryTerminateWorker() {
            synchronized(workers) {
                // Make sure we're not trying race with termination of scheduler
                if (isTerminated) return
                // Someone else terminated, bail out
                if (createdWorkers <= corePoolSize) return
                /*
                 * See tryUnpark for state reasoning.
                 * If this CAS fails, then we were successfully unparked by other worker and cannot terminate.
                 */
                if (!workerCtl.compareAndSet(PARKED, TERMINATED)) return
                /*
                 * At this point this thread is no longer considered as usable for scheduling.
                 * We need multi-step choreography to reindex workers.
                 *
                 * 1) Read current worker's index and reset it to zero.
                 */
                val oldIndex = indexInArray
                indexInArray = 0
                /*
                 * Now this worker cannot become the top of parkedWorkersStack, but it can
                 * still be at the stack top via oldIndex.
                 *
                 * 2) Update top of stack if it was pointing to oldIndex and make sure no
                 *    pending push/pop operation that might have already retrieved oldIndex could complete.
                 */
                parkedWorkersStackTopUpdate(this, oldIndex, 0)
                /*
                 * 3) Move last worker into an index in array that was previously occupied by this worker,
                 *    if last worker was a different one (sic!).
                 */
                val lastIndex = decrementCreatedWorkers()
                if (lastIndex != oldIndex) {
                    val lastWorker = workers[lastIndex]!!
                    workers[oldIndex] = lastWorker
                    lastWorker.indexInArray = oldIndex
                    /*
                     * Now lastWorker is available at both indices in the array, but it can
                     * still be at the stack top on via its lastIndex
                     *
                     * 4) Update top of stack lastIndex -> oldIndex and make sure no
                     *    pending push/pop operation that might have already retrieved lastIndex could complete.
                     */
                    parkedWorkersStackTopUpdate(lastWorker, lastIndex, oldIndex)
                }
                /*
                 * 5) It is safe to clear reference from workers array now.
                 */
                workers[lastIndex] = null
            }
            state = WorkerState.TERMINATED
        }

        // It is invoked by this worker when it finds a task
        private fun idleReset(mode: Int) {
            terminationDeadline = 0L // reset deadline for termination
            if (state == WorkerState.PARKING) {
                assert { mode == TASK_PROBABLY_BLOCKING }
                state = WorkerState.BLOCKING
            }
        }

        // 给我找食物啊
        fun findTask(scanLocalQueue: Boolean): Task? {
            if (tryAcquireCpuPermit()) return findAnyTask(scanLocalQueue)
            // If we can't acquire a CPU permit -- attempt to find blocking task
            val task = if (scanLocalQueue) {
                localQueue.poll() ?: globalBlockingQueue.removeFirstOrNull()
            } else {
                globalBlockingQueue.removeFirstOrNull()
            }
            return task ?: trySteal(blockingOnly = true)
        }

        // 找任何的 task 给我吃
        private fun findAnyTask(scanLocalQueue: Boolean): Task? {
            /*
             * Anti-starvation mechanism: probabilistically poll either local
             * or global queue to ensure progress for both external and internal tasks.
             */
            if (scanLocalQueue) {
                val globalFirst = nextInt(2 * corePoolSize) == 0
                if (globalFirst) pollGlobalQueues()?.let { return it }
                localQueue.poll()?.let { return it }
                if (!globalFirst) pollGlobalQueues()?.let { return it }
            } else {
                pollGlobalQueues()?.let { return it }
            }
            return trySteal(blockingOnly = false)
        }

        // 为第二种 找食物的 task 提供原料
        private fun pollGlobalQueues(): Task? {
            if (nextInt(2) == 0) {
                globalCpuQueue.removeFirstOrNull()?.let { return it }
                return globalBlockingQueue.removeFirstOrNull()
            } else {
                globalBlockingQueue.removeFirstOrNull()?.let { return it }
                return globalCpuQueue.removeFirstOrNull()
            }
        }

        private fun trySteal(blockingOnly: Boolean): Task? {
            assert { localQueue.size == 0 }
            val created = createdWorkers
            // 0 to await an initialization and 1 to avoid excess stealing on single-core machines
            // 核心数少于 2 没有几个 Thread ，我没地方偷东西吃呀
            if (created < 2) {
                return null
            }

            var currentIndex = nextInt(created)
            var minDelay = Long.MAX_VALUE
            repeat(created) {
                ++currentIndex
                if (currentIndex > created) currentIndex = 1
                // 从 魔鬼从中顺序选择
                val worker = workers[currentIndex]
                if (worker !== null && worker !== this) {
                    assert { localQueue.size == 0 }
                    // 我是偷 blocking 的吃还是偷别的捏
                    val stealResult = if (blockingOnly) {
                        // WorkQueue 里面的 tryStealBlockingFrom 方法
                        localQueue.tryStealBlockingFrom(victim = worker.localQueue)
                    } else {
                        localQueue.tryStealFrom(victim = worker.localQueue)
                    }
                    if (stealResult == TASK_STOLEN) {
                        // 偷成功了 那就从 queue 里面取出来给我吃呀嘿嘿嘿
                        return localQueue.poll()
                    } else if (stealResult > 0) {
                        // delay 多久再去偷
                        minDelay = min(minDelay, stealResult)
                    }
                }
            }
            minDelayUntilStealableTaskNs = if (minDelay != Long.MAX_VALUE) minDelay else 0
            return null
        }
    }

```
.1
.1
##### 未完待续的分析
 .1
 .1


LockFreeTaskQueue

WorkingQueue

state

localQueue

indexInArray
.1
.1
##### 核心
.1
核心是 atomic workerCtl 存储状态，以及改变状态。

2
1
##### 管道失序

在 linux 或者 mac 下使用管道之后在使用 grep 会发现内容不仅仅是 history 的会被 grep，全部文件都会，原因是我在 grep 的时候增加了 -rns 参数。
