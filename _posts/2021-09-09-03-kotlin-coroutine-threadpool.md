---
layout: post
title: kotlin 协程的底层支持之线程池(2)
subtitle: 协程中的线程池
image: /img/lifedoc/jijiji.jpg
tags: [ThreadPool]
---

### 协程线程魔鬼的伙伴们

LockFreeTaskQueue ： 这玩意负责魔鬼们的食物池子，这里因为是多个魔鬼消费者，所以这里是要加锁的，内部核心是 LockFreeTaskQueueCore 干的活

WorkingQueue：魔鬼们的口袋实现者，通过这个可以进行偷食物啥的，都是他在辅助饥饿的魔鬼们

state：魔鬼的状态好不好

localQueue：魔鬼的口袋

indexInArray：魔鬼编号 007

| 主角们 | 功效解读 |
|---|---|
| LockFreeTaskQueue | 作为 CoroutineScheduler 的 global queue 存在，魔鬼的食物池子 |
| WorkingQueue | 魔鬼们的缓存池，乌拉 |
| state | 魔鬼现实状态 |
| localQueue | 魔鬼利用 WorkingQueue 本地起得名字 |
| indexInArray | 魔鬼几号，这个是在调用的地方使用 atomic 保证递增的 |

```kotlin

//CoroutineScheduler.dispatch -> 

// 首先尝试找到当前的 worker
val currentWorker = currentWorker()
// 首先尝试将 task 装进 worker 的袋子里（localQueue）
val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)
// 如果没有找到魔鬼，那么就先去把这个食物扔到池子中有魔鬼的时候再给他吃
if (notAdded != null) {
    if (!addToGlobalQueue(notAdded)) {
        throw RejectedExecutionException("$schedulerName was terminated")
    }
}

// Worker.findTask ->
        fun findTask(scanLocalQueue: Boolean): Task? {
            // 如果这里获取到了 cpu 就去 findAnyTask
            if (tryAcquireCpuPermit()) return findAnyTask(scanLocalQueue)
            // If we can't acquire a CPU permit -- attempt to find blocking task
            // 如果优先使用自己的就搞自己，否则从食物池子取
            val task = if (scanLocalQueue) {
                localQueue.poll() ?: globalBlockingQueue.removeFirstOrNull()
            } else {
                // 直接从食物池子里面搞
                globalBlockingQueue.removeFirstOrNull()
            }
            // 还没有搞到，我踏马直接去偷别的魔鬼的
            return task ?: trySteal(blockingOnly = true)
        }


// findAnyTask
        private fun findAnyTask(scanLocalQueue: Boolean): Task? {
            if (scanLocalQueue) {
                val globalFirst = nextInt(2 * corePoolSize) == 0
                if (globalFirst) pollGlobalQueues()?.let { return it }
                // 优先自己的口袋里取还是从食物池子里搞
                localQueue.poll()?.let { return it }
                if (!globalFirst) pollGlobalQueues()?.let { return it }
            } else {
                // 二话不说食物池子里面搞
                pollGlobalQueues()?.let { return it }
            }
            // 没找到我踏马偷
            return trySteal(blockingOnly = false)
        }

// pollGlobalQueues 搞同步的还是异步的
        private fun pollGlobalQueues(): Task? {
            if (nextInt(2) == 0) {
                globalCpuQueue.removeFirstOrNull()?.let { return it }
                return globalBlockingQueue.removeFirstOrNull()
            } else {
                globalBlockingQueue.removeFirstOrNull()?.let { return it }
                return globalCpuQueue.removeFirstOrNull()
            }
        }

// trySteal 我踏马偷别人的吃，乌拉
        private fun trySteal(blockingOnly: Boolean): Task? {
            assert { localQueue.size == 0 }
            val created = createdWorkers
            // 只有特么我自己，偷个毛线
            if (created < 2) {
                return null
            }

            var currentIndex = nextInt(created)
            var minDelay = Long.MAX_VALUE
            // 从第一个开始吧老弟
            repeat(created) {
                ++currentIndex
                if (currentIndex > created) currentIndex = 1
                // 遍历魔鬼们
                val worker = workers[currentIndex]
                if (worker !== null && worker !== this) {
                    assert { localQueue.size == 0 }
                    // 我是从阻塞的偷还是不阻塞的偷呢
                    val stealResult = if (blockingOnly) {
                        localQueue.tryStealBlockingFrom(victim = worker.localQueue)
                    } else {
                        localQueue.tryStealFrom(victim = worker.localQueue)
                    }
                    // 偷到了！！！！
                    if (stealResult == TASK_STOLEN) {
                        return localQueue.poll()
                    } else if (stealResult > 0) {
                        // 木有偷到，那么就特么等等再说
                        minDelay = min(minDelay, stealResult)
                    }
                }
            }
            // park 一会，一会再回来搞吃滴，这个参数是别的地方使用的
            minDelayUntilStealableTaskNs = if (minDelay != Long.MAX_VALUE) minDelay else 0
            return null
        }
```
