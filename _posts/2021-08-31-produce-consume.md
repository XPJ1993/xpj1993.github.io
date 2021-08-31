---
layout: post
title: 生产者消费者模式
subtitle: 该模式一般作为多线程共享资源使用
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---


| 生产者模式 | 常用方式 |
| --- | --- |
| 使用 BlockingQueue | 使用 BlockingQueue 核心是多个生产者和消费者共用一个 queue 生产者去阻塞 offer （offer 可以添加超时机制，一直无人消费就丢弃）， 消费者去 take|
| 使用 ReentrantLock 配合 Condition | 消费者与生产者共享一把 ReentrantLock 然后使用不同的 Condition 作为触发条件，消费者在去消费的时候发现数据为空 则使用 empty Condition 发信号，并且在 full Condition await，反之亦然 |


**使用 Condition 的用法**

```java


package com.xpj.javagrowth.produce;

/**
 * author : xpj
 * date : 8/31/21 4:41 PM
 * description :
 */


import java.util.List;
import java.util.Random;

public class TestProduceAndConsumer {

    public static void PRTMsg(String msg) {
        System.out.println("CURRENT " + Thread.currentThread().getName() +
                " TIME " + System.currentTimeMillis() + " THREAD ID: " + Thread.currentThread().getId()
                + " MSG -> " + msg);
    }

    /**
     * 消费者
     */
    public static class Consumer implements Runnable {
        private List<PCData> queue;

        public Consumer(List<PCData> queue) {
            this.queue = queue;
        }

        @Override
        public void run() {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    PCData data = null;
                    // 核心是这里都需要先加锁
                    Main.INSTANCE.getLock().lock();
                    PRTMsg("++++消费者ID: size " + queue.size());
                    // 这里确认 queue 里面没有数据了，那么就会给 empty condition 发送信号，说我没有数据了，
                    // 来生产呀
                    if (queue.size() == 0) {
                        Main.INSTANCE.getEmpty().signal();
                        Main.INSTANCE.getFull().await();
                    }
                    PRTMsg("++++++++++++消费者ID AFTER:------");
                    Thread.sleep(300);
                    data = queue.remove(0);
                    // 处理完毕之后释放锁
                    Main.INSTANCE.getLock().unlock();
                    PRTMsg(">>>>>>>>>>>>>消费者ID:消费了:" + data.getData());
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }


    /**
     * 生产者
     */
    public static class Producter implements Runnable {
        private List<PCData> queue;
        private int len;

        public Producter(List<PCData> queue, int len) {
            this.queue = queue;
            this.len = len;
        }

        @Override
        public void run() {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    Random r = new Random();
                    PCData data = new PCData();
                    data.setData(r.nextInt(500));
                    Main.INSTANCE.getLock().lock();
                    PRTMsg("生产者ID: size : " + queue.size());
                    // 增加这个是为了 queue 里面有数据就通知 消费者，不积压数据
                    if (queue.size() != 0) {
                        Main.INSTANCE.getFull().signal();
                        PRTMsg("NOTIFY +++++ 生产者ID: size : " + queue.size());
                    }
                    // 这里数据达到上限了会通知 full condition 已经满了，来消费吧，同时 empty 等待
                    if (queue.size() >= len) {
                        Main.INSTANCE.getFull().signal();
                        Main.INSTANCE.getEmpty().await();
                    }
                    Thread.sleep(330);
                    queue.add(data);
                    Main.INSTANCE.getLock().unlock();
                    PRTMsg("<<<<<<生产者ID:  生产了:" + data.getData());
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }


    public static class PCData {
        private int data;

        public int getData() {
            return data;
        }

        public void setData(int data) {
            this.data = data;
        }
    }
}


```

**使用 BlockingQueue 的生产者与消费者模式**

```kotlin
package com.xpj.javagrowth.produce

import java.lang.Exception
import java.util.concurrent.BlockingQueue
import java.util.concurrent.TimeUnit
import kotlin.random.Random

/**
 * author : xpj
 * date : 8/31/21 6:44 PM
 * description :
 */
class TestProduceUseBlock {

}

class Produce(val blockQueue: BlockingQueue<TestProduceAndConsumer.PCData>) : Runnable {
    override fun run() {
        var data: TestProduceAndConsumer.PCData?
        val random = Random(System.currentTimeMillis())
        try {
            while (!Thread.currentThread().isInterrupted) {
                Thread.sleep(300)
                data = TestProduceAndConsumer.PCData()
                data.data = random.nextInt(10000)
                // 这里尝试插入数据到 array blockingqueue 中，如果此时满了就超时 1s ，之后如果还插入不了就失败
                if (!blockQueue.offer(data, 1, TimeUnit.SECONDS)) {
                    TestProduceAndConsumer.PRTMsg("insert data fail. ${blockQueue.size}")
                } else {
                    TestProduceAndConsumer.PRTMsg(" insert data success ${data.data}")
                }
            }
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

}

class Consumer1(val blockQueue: BlockingQueue<TestProduceAndConsumer.PCData>) : Runnable {
    override fun run() {
        try {
            while (!Thread.currentThread().isInterrupted) {
                // 每 900ms 消费一条数据，这里是为了营造生产者过量生产的场景，
                Thread.sleep(900)
                // take 之后 queue 会 size 减一
                val data = blockQueue.take()
                if (data != null) {
                    TestProduceAndConsumer.PRTMsg(" consumer1 take value is ${data.data}")
                }
            }
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
}
```

**测试驱动代码**：

```kotlin
package com.xpj.javagrowth.produce

import com.xpj.javagrowth.produce.TestProduceAndConsumer.*
import java.util.*
import java.util.concurrent.ArrayBlockingQueue
import java.util.concurrent.Executors
import java.util.concurrent.locks.ReentrantLock
import kotlin.system.exitProcess

/**
 * author : xpj
 * date : 8/31/21 4:43 PM
 * description :
 */

object Main {
    var lock = ReentrantLock()
    var empty = lock.newCondition()
    var full = lock.newCondition()
    @JvmStatic
    fun main(args: Array<String>) {
        if (true) {
            return main1()
        }
        val queue: List<PCData> = ArrayList()
        val length = 10
        val p1 = Producter(queue, length)
        val p2 = Producter(queue, length)
        val p3 = Producter(queue, length)
        val c1 = Consumer(queue)
        val c2 = Consumer(queue)
//        val c3 = Consumer(queue)
        // 使用这种方式生成的 ThreadPool 会持续的创建新的线程
        val service = Executors.newCachedThreadPool()
        service.execute(p1)
        service.execute(p2)
        service.execute(p3)
        service.execute(c1)
        service.execute(c2)
        val scanner = Scanner(System.`in`)
        while (true) {
            val quit = scanner.nextInt()
            if (quit == 1) {
                exitProcess(1)
            }
        }

    }

    fun main1() {
        PRTMsg("Test use blocking queue")
        val queue = ArrayBlockingQueue<PCData>(10)
        val p1 = Produce(queue)
        val p2 = Produce(queue)
        val p3 = Produce(queue)
        val c1 = Consumer1(queue)
        val c2 = Consumer1(queue)
        val service = Executors.newCachedThreadPool()
        service.execute(p1)
        service.execute(p2)
        service.execute(p3)
        service.execute(c1)
        service.execute(c2)
        val scanner = Scanner(System.`in`)
        while (true) {
            val quit = scanner.nextInt()
            if (quit == 1) {
                exitProcess(1)
            }
        }
    }
}

```

| 1 | 2 |
| --- | --- |
| hello : world | h: d | 
| hello | baby |
