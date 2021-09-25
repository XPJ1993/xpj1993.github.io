---
layout: post
title: 最大 K 个数
subtitle: 借助 Java PriorityQueue
image: /img/lifedoc/jijiji.jpg
tags: [Algorithm]
---

sheet

| 数组中找到最大 / 小 的K个数 | 解法详情 |
|---|---|
| 找最大 K 个数 | 使用小根堆去解决，然后 Java 里面有一个数据结构 PriorityQueue 就是默认的小根堆实现，所以找出 n 个数据的数组中最大的 K 个数就用这个数据机构就可以了，核心点就是小于 K 直接输入，大于 K 的则需要判断新的值是否大于这个根，如果大于，那么就把根丢掉，也就是取出来置为 null |
| 找最小 K 个数 | 这个明显是可以使用大根堆去解决的，但是没有现成的大跟堆数据结构，但是人家数据结构定义的比较好，就是在构造的时候也可以通过重载方法传入对应的 comparator 去定义反过来的大根堆，这里的核心区别就是一个是需要构造大跟堆，一个是判断需要比 ROOT 小，因为是最小的几个数么 |

```kotlin
package com.xpj.algo

import java.util.*

/**
 * @author xuesheng
 * @date 2021/9/25
 * @description 使用小跟堆解决一个 n 数组中的最大 k 个数，使用大跟推解决 n 数组中 最小 k 个数
 **/

private fun createIntArray(size: Int): IntArray {
    val rand = getRandom()
    val array = IntArray(size)
    repeat(size) {
        array[it] = rand.nextInt(10000)
    }
    return array
}

// use priority queue process big k problem.
fun findBiggestKNum(n: Int, k: Int) {
    val initData = createIntArray(n)
    ALGPrint("init data is : ")
    initData.forEach {
        ALGPrintSameLine(it)
    }
    ALGPrintSameLine("\n")
    // default is small root heap. if queue size more than k, compare root and want
    // insert data .
    val queue = PriorityQueue<Int>(k)
    for (i in initData) {
        if (queue.size != k) {
            // add same to offer.
            queue.offer(i)
        } else if (queue.peek() < i) { // 注意这里是 小于 i
            // 1. poll or remove first. and insert i
            var del = queue.poll()
            // 2. del to null.
            del = null
            // 3. insert i to queue
            queue.offer(i)
        }
    }

    ALGPrint("queue is : $queue")
}

fun findSmallestKNum(total : Int, k: Int) {
    if (total <= 0 || k <= 0 || total < k) {
        return
    }
    if (total == k) {
        ALGPrint("the same.")
        return
    }

    val initData = createIntArray(total)
    //   注意 这里传入了 自定义 comparator comparator 这里构造一个大根堆
    // 那么我们每次传进去的值使用最大值判断就行了，如果 root 大于 新的值，那么我们就置换
    // 注意 这里不满足 k 的时候直接 offer 就行了，其他的需要 判断 置换先 poll 出来设置为空在换
    // 保证 队列 长度不要超过 k

    val queue = PriorityQueue<Int>(k) { o1, o2 -> o2!! - o1!! }
    ALGPrint("smallest init data is : ")
    initData.forEach {
        ALGPrintSameLine(it)
    }
    ALGPrintSameLine("\n")
    for (i in initData) {
        if (queue.size != k) {
            // add same to offer.
            queue.offer(i)
        } else if (queue.peek() > i) { // 注意这里是 大于 i
            // 1. poll or remove first. and insert i
            var del = queue.poll()
            // 2. del to null.
            del = null
            // 3. insert i to queue
            queue.offer(i)
        }
    }

    ALGPrint("smallest queue is : $queue")
}

```

PriorityQueue 里面的核心排序部分：
内部是一个完全二叉树，根据那个 comparator 去控制大跟还是小跟，默认小跟。

```kotlin
// 在调用 offer 的时候首先判断如果没有元素就放到第一个位置里
if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e); // 这里的 i 是 size + 1，所以在下面的 sift 方法会 先 - 1
// 这里如果不是第一个就调用 siftUp 方法
if (comparator != null)
            siftUpUsingComparator(k, x);
        else
            siftUpComparable(k, x);
// 我们可以先看 comparator 的方法 siftUpComparable
    @SuppressWarnings("unchecked")
    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            // 首先找到 parent 位置，这里是从叶子往上找，这里为什么是 k-1 呢，因为 4、5 的父节点都是 3，减了也可以
            int parent = (k - 1) >>> 1;
            // 拿到父节点
            Object e = queue[parent];
            // 判断要传入的值是否大于 父节点，如果大于那么我们就找到位置了
            if (key.compareTo((E) e) >= 0)
                break;
            // 没找到，就把 e 放到最大的那个位置
            queue[k] = e;
            // 更新 k 为 parent 位置
            k = parent;
        }
        // 这里把这个值放到对的位置
        queue[k] = key;
    }
// 再看一下自定义的 comparator 的情况，与上面代码雷同，主要就是使用 comparator 提供一个达到 break 的条件
    @SuppressWarnings("unchecked")
    private void siftUpUsingComparator(int k, E x) {
        while (k > 0) {
            /*
            java提供两种右移运算符，属于位运算符。位运算符用来对二进制位进行操作。
>>  ：算术右移运算符，也称带符号右移。用最高位填充移位后左侧的空位。
>>>：逻辑右移运算符，也称无符号右移。只对位进行操作，用0填充左侧的空位。
            */
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (comparator.compare(x, (E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = x;
    }

```

**总结**：
小根堆或者大根堆只会保证根必然小于或者大于下面的子节点，堆整体的顺序是没有保证的，因为是基于完全二叉树的，所以每次无符号右移一位找到父节点，和父节点比较就可以了，这样的话可以弄成阶乘下降的查找，效率大大提升，同样地，这样也能保证优先消费比较大的，也就是他们的叶子节点，我们无序数组中找 K 大或者小的数就是利用的这个完全二叉树的小根堆的特性，实际上是一个数组在存着。实际业务中如果真的有这种优先级或者 K 大或者小的都可以用这个基于完全二叉树的 大/小 根堆去完成。
