---
layout: post
title: 享元模式
subtitle: 享元模式解析和使用场景
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---

享元模式的使用和解析。

| 享元模式 | 常用方式及场景 |
|---|---|
| 常用方式 | 一般享元模式都是配合池化技术相互配合去使用，这种是为了节省内存空间，以及分配的时间 |
| 使用场景 | 有很多需要使用的复杂占用内存的对象，使用池化技术去缓存以节约分配时间以及内存 |
| Android 使用方式 | Handler 里面 Message 使用一个静态链表去共享这个 Message 池子的，当调用 Message.obtain() 方法去申请释放这样的话就可以复用他之前分配的各种东西了 |

Message 含有 13 个局部变量，这导致每次分配无论空间以及时间复杂度上都是比较可观的。

```java
int what;
int arg1;
int arg2;
Object obj;
Messenger replayTo;
Long sendingUid;
Long workSourceUid;
int Flags;
Long when;
Bundle data;
Handler target;
Runnable callback;
Message next；
```

代码：


```java
/*
1. 抢到锁
2. sPool 不为空，头节点
3. 取到头节点然后 sPool 往后推一位，并且将取出来的和后面的链断开
4. 初始化 flags
5. 已经池化的 减一
6. 返回分配好的 Message
7. 池子里面没有，直接返回 new 出来的对象
*/
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

/*
回收
1. 清空标志位以及自带各种的为初始值
2. 同步 sPoolSync 锁
3. 获取到锁之后判断小于 POOL 50 长度，设置自己的 next 为之前的 头节点，然后 sPool 更新头节点为当前已经清空所有状态的 Message
*/
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }  


/*
Message 是何时调用的 recycle 去回收到池子里面呢，咱们从 Handler 机制里面去探索下。

这里去添加的时候判断，如果这会正在推出，那么就会把这些已经分配好的去缓存到池子里面去，一般情况下我们不需要去主动 recycle 这些都是系统不同的使用地方帮咱们完成的。
*/

boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }

        synchronized (this) {
            if (msg.isInUse()) {
                throw new IllegalStateException(msg + " This message is already in use.");
            }

            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
        // ...     
    }
}

/*
Hanlder 机制里面 消息的驱动流程为 epoll 阻塞去控制节奏，有几个触发点，一个是咱们 post 或者 sendMsg 的时候去 wakeup 另一个就是咱们 wakeUp 的时候判断 current time 小于 取出来的 msg.when 那么就会调用 native 去阻塞，另外就是 MessageQueue 里面第一时间去拿到 native 的 long 句柄。释放的时候是在 finalize 里面释放句柄的。
*/

Handler() {
    // 初始化的时候会先拿本身的 Looper 如果线程没有准备过 Looper 就会抛出异常。
    mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
    }
}
```

### 总结

享元模式翻译过来就是共享元素的意思，标准的享元模式里有共享以及不共享的内容，我们 Message 这里所有的都是可以共享的，享元模式一般就是用于节省空间及时间的分配，然后他们之间都有很多共有的元素。

