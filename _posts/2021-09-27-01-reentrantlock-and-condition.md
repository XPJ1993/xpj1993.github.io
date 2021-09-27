---
layout: post
title: Reentrant Lock 和 Condition
subtitle: reentrant lock 是如何使用 condition 细粒度管理线程同步
image: /img/lifedoc/jijiji.jpg
tags: [Java]
---

Reentrant 与 Condition 解析

| 使用场景 | 使用方式 |
|---|---|
| 可中断的场景 | Synchronized 是不可中断的，所以如果某些线程抢到时间片之后但是还是可以被中断，那么请选择 ReentrantLock |
| 唤醒特定线程 | 可以从一个 ReentrantLock 里面获取对应的 Condition 去做到具体的线程唤醒和阻塞，而 Synchronized 只能通过 object.notify / notifyAll 唤醒随机和所有线程，一个是导致争抢，一个是导致惊群 |
| 是否可重入 | ReentrantLock 顾名思义可重入 |
| 是否公平 | ReentrantLock 可以在构造的时候指定是否是公平锁 |

现在就来解释下为什么用 ReentrantLock 配合 Condition 可以达到交替打印的目的，关于，交替打印我好像也高了一个协程的，等和 Java 那版本放一起搞，协程这个是基于 Channel 的，因为协程之间使用 Channel 通信是公平的，比这个还好控制些呢，Java 版本的需要 volitail 去控制共同改变的变量。


解析 ReentrantLock 是怎样配合 Condition 完成细粒度控制的，解析 ReentrantLock 公平锁与非公平锁的区别。

```java
// 几个常用方法
ReentrantLock lock = new ReentrantLock(false);
    Condition condi = lock.newCondition();

    void func1() {
        condi.await();
        condi.signal();
    }

// 是否公平锁在于构造参数传入的数值，这里的区别就是 new 的 FairSync 还是 NonfairSync， NofairSync 就是默认的继承 Sync 就可以，然后 Sync 继承了 AbstractQueuedSynchronizer ，然后实现了 nonfairTryAcquire
sync = fair ? new FairSync() : new NonfairSync();
// Fair 和 Nonfair 的区别
protected final boolean tryAcquire(int acquires) { // Nonfair 直接调用父类的 nonfairTryAcquires
            return this.nonfairTryAcquire(acquires);
        }

        final boolean nonfairTryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = this.getState();
            if (c == 0) {
                if (this.compareAndSetState(0, acquires)) {
                    this.setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == this.getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) {
                    throw new Error("Maximum lock count exceeded");
                }

                this.setState(nextc);
                return true;
            }

            return false;
        }        
// 这里去实现公平锁自己的
protected final boolean tryAcquire(int acquires) {
    // ...
    Thread current = Thread.currentThread();
            int c = this.getState();
            if (c == 0) {
                // 多了一个 hasQueuedPredecessors 判断条件，这里主要就是遍历从后往前遍历节点，看看有没有比自己更新进入队列的，如果有的话就返回 true ，那么这里就不争抢了
                // 后面都是使用 cas 去尝试设置进去，然后设置 这个锁目前的持有者是自己
                if (!this.hasQueuedPredecessors() && this.compareAndSetState(0, acquires)) {
                    this.setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == this.getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) {
                    throw new Error("Maximum lock count exceeded");
                }

                this.setState(nextc);
                return true;
            }

            return false;
}

// 看一下 ReentrantLock 的 lock 和 unlock 方法，看看是怎么实现可重入的，应该就是计数
    public void lock() {
        this.sync.acquire(1);
    }

        public void unlock() {
        this.sync.release(1);
    } // 核心就是堆这个 加一 和 减一

// 看看 tryLock 和 其他
    public void lockInterruptibly() throws InterruptedException {
        this.sync.acquireInterruptibly(1);
    }

    public boolean tryLock() {
        // 难怪把这个 nonfair 方法放到父类，因为 tryLock 的时候都是调用的这个
        return this.sync.nonfairTryAcquire(1);
    }

    // 超时获取失败就算了的锁
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return this.sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

// ReentrantLock 的 newCondition，底层是调用的 new ConditionObject
final ConditionObject newCondition() {
            return new ConditionObject(this);
        }

// Condition 的 await 和 signal
        public final void await() throws InterruptedException {
            if (Thread.interrupted()) {
                throw new InterruptedException();
            } else {
                // 主要逻辑在 Node 中，这里 addConditionWaiter 是核心，把当前线程构造一个装进去
                /*
                构建一个 Node
                */
                AbstractQueuedSynchronizer.Node node = this.addConditionWaiter();
                int savedState = AbstractQueuedSynchronizer.this.fullyRelease(node);
                int interruptMode = 0;

                while(!AbstractQueuedSynchronizer.this.isOnSyncQueue(node)) {
                    LockSupport.park(this); // park 住了
                    if ((interruptMode = this.checkInterruptWhileWaiting(node)) != 0) {
                        break;
                    }
                }

                if (AbstractQueuedSynchronizer.this.acquireQueued(node, savedState) && interruptMode != -1) {
                    interruptMode = 1;
                }

                if (node.nextWaiter != null) {
                    this.unlinkCancelledWaiters();
                }

                if (interruptMode != 0) {
                    this.reportInterruptAfterWait(interruptMode);
                }

            }
        }

        private void doSignal(AbstractQueuedSynchronizer.Node first) {
            // 唤醒第一个等待的节点
            do {
                if ((this.firstWaiter = first.nextWaiter) == null) {
                    this.lastWaiter = null;
                }

                first.nextWaiter = null;
            } while(!AbstractQueuedSynchronizer.this.transferForSignal(first) && (first = this.firstWaiter) != null);

        }
// addConditionWaiter 方法
private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }

            Node node = new Node(Node.CONDITION);
}        
// node 的构造方法,把当前 Thread 放到 Node 里面
        Node(int waitStatus) {
            U.putInt(this, WAITSTATUS, waitStatus);
            U.putObject(this, THREAD, Thread.currentThread());
        }        
```


