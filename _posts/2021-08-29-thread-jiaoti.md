---
layout: post
title: Java 线程交替打印
subtitle: 使用 ReentrantLock 及 Condition 实现
image: /img/lifedoc/jijiji.jpg
tags: [Java]
---

经典面试题，使用线程实现交替打印，这里是使用的 ReentrantLock 可重入锁配合 Condition 实现交替打印。

```java
package com.xpj.kotlingrowth.thread;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * author : xpj
 * date : 8/28/21 6:20 PM
 * description :
 */
public class ABCTestNum extends Thread {
    /**
     * 多个线程共享这一个sequence数据
     */
    private static volatile int num = 0;
    // 使用 volatile 保证多线程的可见性

    private static final int SEQUENCE_END = 15;

    private final Integer id;
    private ReentrantLock lock;
    private Condition[] conditions;


    ABCTestNum(Integer id, ReentrantLock lock, Condition[] conditions) {
        this.id = id;
        this.setName("我是Thread " + id + " 号");
        this.lock = lock;
        this.conditions = conditions;
    }

    @Override
    public void run() {
        // num 这里使用 volatile 可以直接不进入 while 循环
        while (num >= 0 && num < SEQUENCE_END) {
            // 加锁
            lock.lock();
            try {
                // todo 加上这句才能生效，需要在 num 上加上 volatile 关键字在几个线程之间透明共享
                if (num >= SEQUENCE_END) {
                    System.out.println("哈哈哈哈哈，啊啊啊，我到了要结束的时候了 " + id + " num : " + num);
                    break;
                }
                Thread.sleep(200);
                //对序号取模,如果不等于当前线程的id,则先唤醒其他线程,然后当前线程进入等待状态
                // 这里是核心，公有的数对数量取模，如果等于自己的 id 那么就不进入这个循环，否则就唤醒 +1 的那个 condition，这里不使用 while 使用 if 也可以，自己 await 就不会打印下面的了
                while (num % conditions.length != id) {
                    System.out.println("啊啊啊，我这里的 " +id + " 不对呀，唤醒别人吧。整除唤醒啊。");
                    // 召唤 id + 1 的 condition ，给他发信号
                    conditions[(id + 1) % conditions.length].signal();
                    // 自己进入 await 状态
                    conditions[id].await();
                }
                System.out.println(Thread.currentThread().getName() + " 打印 ==》 " + num);
                //序号加1
                num = num + 1;
                //唤醒当前线程的下一个线程
                conditions[(id + 1) % conditions.length].signal();
                //当前线程进入等待状态
                conditions[id].await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                //将释放锁的操作放到finally代码块中,保证锁一定会释放
                lock.unlock();
            }
        }
        System.out.println("呜啦啦啦啦，磨磨唧唧，我爱你，啊啊啊，我到了要结束的时候了 " + id);
        //数字打印完毕,线程结束前唤醒其余的线程,让其他线程也可以结束
        end();
    }

    // 第一个走到这里的线程已经 died 了， 因此别的再去发送 signal 也木有作用了，因为承载的线程已经 died 了呀。
    private void end() {
        System.out.println("收尾，但是也是这里导致的多打数字吗 " + id);
        lock.lock();
        int endSize = conditions.length - 1;
        for (int i = 1; i<= endSize; i++) {
            conditions[(id + i) % conditions.length].signal();
        }
        lock.unlock();
        System.out.println("这里是收尾，after unlock " + id);
    }

}


```


```kotlin
package com.xpj.kotlingrowth.thread

import java.util.concurrent.locks.Condition
import java.util.concurrent.locks.ReentrantLock

/**
 * author : xpj
 * date : 8/28/21 6:21 PM
 * description :
 */


fun main() {
    val threadCount = 4
    // 使用共同的锁 ReentrantLock 可重入锁
    val lock = ReentrantLock()
    // 这里根据有几个线程创建几个 condition
    val conditions = arrayOfNulls<Condition>(threadCount)
    for (i in 0 until threadCount) {
        conditions[i] = lock.newCondition()
    }
    // 创建对应数量的线程类
    val printNumbers = arrayOfNulls<ABCTestNum>(threadCount)
    for (i in 0 until threadCount) {
        printNumbers[i] = ABCTestNum(i, lock, conditions)
    }
    // 启动线程
    for (printNumber in printNumbers) {
        printNumber!!.start()
    }
}

```
