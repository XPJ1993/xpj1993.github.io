---
layout: post
title: 二叉树的遍历
subtitle: 前、中、后序遍历，主要是针对的 Root 节点来说
image: /img/lifedoc/jijiji.jpg
tags: [Algorithm]
---

二叉树的遍历，首先是写出二叉树的三种常见遍历。

三种常见遍历方法为，前中后，这些遍历方法的定义主要是基于对 Root 节点而言的，**前序**的定义是首先输出 Root 节点，然后左右子节点。**中序**的含义是，左 Root 右。**后序**是，左 右 Root 这个顺序。可以看到，所谓的前中后实际上是 Root 节点的是处在 前、中与后，左右子节点是固定的。

| 遍历方法 | 定义极其实现 |
|---|---|
| 前序遍历 | 首先访问 Root， 然后递归调用方法访问左节点，右节点 |
| 中序遍历 | 首先访问左节点，然后访问 Root ，再然后访问右节点 |
| 后序遍历 | 首先访问左节点，然后右节点，然后是 Root |

树的构造，我这了是生成随机数然后判断是否为 偶数 ，然后赋值给左节点或者右节点的，这种方式构造的树只会在一支上延伸下去。

树节点代码以及三种遍历：

```kotlin
class TreeNode constructor(val value: Int) {
    // 二叉树的定义，拥有左右子树
    var left: TreeNode? = null
    var right: TreeNode? = null

    companion object {
        // 前序遍历，优先跟节点，然后左，然后右
        fun preOrderPrint(r: TreeNode?, list:MutableList<Int>) {
            if (r == null) {
                return
            }
            list.add(r.value)
            ALGPrint("pre order is: $r ")
            preOrderPrint(r.left, list)
            preOrderPrint(r.right, list)
        }

        // 中序遍历，优先左节点，然后跟节点，最后右节点
        fun inOrderPrint(root: TreeNode?, list: MutableList<Int>) {
            if (root == null) {
                return
            }
            inOrderPrint(root.left, list)
            list.add(root.value)
            ALGPrint("in order: $root")
            inOrderPrint(root.right, list)

        }

        // 后序遍历，优先左节点，然后右节点，最后根节点
        fun postOrderPrint(root: TreeNode?, list: MutableList<Int>) {
            if (root == null) {
                return
            }
            postOrderPrint(root.left, list)
            postOrderPrint(root.right, list)
            list.add(root.value)
            ALGPrint("in order: $root")
        }
    }

    override fun toString(): String {
        return "current : $value left : ${left?.value} right: ${right?.value}"
    }
}

```

构造树的代码：

```kotlin
    private fun createTreeNode(length: Int): TreeNode {
        val rand = getRandom()
        val root = TreeNode(rand.nextInt(100))
        var p = root
        var isRoot = true
        ALGPrint("root is : $root")
        repeat(length) {
            val num = rand.nextInt(2000)
            if (num % 2 == 0) {
                p.left = TreeNode(num)
            } else {
                p.right = TreeNode(num)
            }

            var next = rand.nextInt(2000)
            if (next == num) {
                next = num + 1
            }
            ALGPrint("it is $it : num is $num ------- next $next")

            val bb = rand.nextInt(3)
            // 这里是增加左右节点，为了构造二叉树，否则不会有俩个叉子，增加这个就会比传入的 length 长
            if (isRoot || (bb != 0 && next % bb == 0)) {
                isRoot = false
                if (p.right == null) {
                    p.right = TreeNode(next)
                } else if (p.left == null) {
                    p.left = TreeNode(next)
                }
            }

            p = setLeftOrRight(next % 2 == 0, p)

        }
        return root
    }

    private fun setLeftOrRight(isFirstLeft: Boolean, root: TreeNode?) : TreeNode{
        var p : TreeNode? = root
        var isSet = false
//        ALGPrint("is first left: $isFirstLeft p is $p")
        if (isFirstLeft) {
            if (p?.left != null) {
                isSet = true
                p = p.left!!
            }
            if (!isSet && p?.right != null) {
                p = p.right!!
            }
        } else {
            if (p?.right != null) {
                isSet = true
                p = p.right!!
            }
            if (!isSet && p?.left != null) {
                p = p.left!!
            }
        }
        return p!!
    }

```

