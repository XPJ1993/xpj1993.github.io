---
layout: post
title: 简单算法学习
subtitle: 冒泡、快排与翻转链表
image: /img/lifedoc/jijiji.jpg
tags: [Algorithm]
---

学习算法，有起码的编码能力。

一直没有刷过算法，现在看来算法还是比较难的，考察编码能力，这个诚不我欺。

| title1 | title2 |
|---|---|
| content1 | content2 |

#### 冒泡排序

这个理论上是最简单的排序算法，但是我第一次也写错了，头脑里肯定知道怎么做，但是写出来味道还是不对，第一次由于没有将内循环里面的起始变量设置为外循环的迭代加一导致乱序。
完整版的代码如下：

```kotlin
    fun bubbleSort(datas: MutableList<Int>) {
        var i = 0
        var jj = 1
        val length = datas.size

        // 这种 for 循环就是这样形式，注意这里使用 until 前开后闭， 使用 .. 是闭区间，这里同样滴
        // inf 从 ind + 1 开始
        for (ind in 0 until length) {
            for (inf in ind + 1 until length) {
                if (datas[ind] > datas[inf]) {
                    val t = datas[ind]
                    datas[ind] = datas[inf]
                    datas[inf] = t
                }
            }
        }

//        while (i < length) {
//            while (jj < length) {
//                if (datas[i] >= datas[jj]) {
//                    val t = datas[i]
//                    datas[i] = datas[jj]
//                    datas[jj] = t
//                }
//
//                jj += 1
////                println("jj is $jj")
//            }
//
////            println("i is $i")
//            i += 1
//            // 缺少这个条件导致排序失败, 因为这里已经有序，如果 jj 不大于 i 会导致循环排序的问题
//            jj = i + 1
//        }
    }
```

#### 快速排序

快速排序的核心是选取任意一个值让他做中间值（大小意义上的），然后从数组或者链表尾部开始往前找，找到比这个中间值小的就退出这个 while ，然后将这个找到的值放到 low 位置，之后再从 low 往上找，找到第一个大于中间值的就退出这个 while 然后交换 high 与这个找到的 值，然后判断 low 是否小于 high，如果大于等于就退出这一次循环，然后递归进行左边排序与右边排序。

```kotlin
    fun quickSort(source: MutableList<Int>, left: Int, right: Int) {
        // 终止条件，right 小于 left
        if (right <= left) return
        // 首先选取最左边的作为中间值，然后首先从右向左找小于中间值的置换到左边
        val spilt = source[left]
        // 最低点选择最左边的值
        var low = left
        // 最高点选择最右值
        var high = right

        // 只要 low != high 那就一直找下去
        while (low < high) {
            // 这一趟找到最右边比中间值小的
            while (low < high && source[high] >= spilt) {
                high -= 1
            }

            // 将这个值赋值到最 low 的地方
            source[low] = source[high]

            // 找到第一个大于中间值的地方
            while (low < high && source[low] <= spilt) {
                low += 1
            }

            // 将这个替换刚刚 high 值的地方
            source[high] = source[low]
        }

        // 将我们中间的值赋值到我们变换后的 起始值 哪里
        source[low] = spilt

        // 递归 起始值为 left 终点 为 low - 1
        quickSort(source, left, low - 1)
        // 递归 起始值为 低值 加一 终点为 right
        quickSort(source, low + 1, right)
    }

```

#### 链表翻转

主要的核心思想是先把链表的每一个节点都放到 stack 中，然后反向取出来塞到新的 head 上，注意这里需要判断 stack size == 0 的时候要切断之前链表最后的联系，使得最后的 next 为 null。

stack 使用 LinkedList 作为实现， removeLast 作为栈的后进先出实现。

```kotlin
    fun reverseListNode() {
        val node: ListNode = createNode(5)
        ALGPrint("before reverse is : ")
        ListNode.printList(node)

        // 栈使用 LinkedList 作为实现
        val stack = LinkedList<ListNode>()
        var nodeTemp: ListNode? = node
        val head = nodeTemp
        nodeTemp = head?.next
        // 第一步添加到 stack 中
        while (nodeTemp != null) {
            stack.add(nodeTemp)
            nodeTemp = nodeTemp.next
        }

        ALGPrint("im break line. stack is $stack")

        val newHead = ListNode()
        var headAfter : ListNode ?= newHead
        while (true) {
            try {
                if (stack.size == 0) {
                    // 需要知道栈里面如果空了的话需要将尾部的 next 指向 null 断开原来 node 的本身的连接
                    headAfter?.next = null
                    break
                }
                ALGPrint("stack size is: ${stack.size}")
                // 取 last 作为栈的后进先出
                headAfter?.next = stack.removeLast()
                headAfter = headAfter?.next
                ALGPrint("after stack size is: ${stack.size}")
            } catch (e: NoSuchElementException) {
                break
            }
        }
        ALGPrint("after reverse is: ")
        ListNode.printList(newHead)
    }
```

