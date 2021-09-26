---
layout: post
title: ThreadLocal 解析
subtitle: ThreadLocal 为本线程保存数据
image: /img/lifedoc/jijiji.jpg
tags: [Java]
---

sheet

| 关键概念 | 这样做的理由 |
|---|---|
| set | 设置一个值到 ThreadLocal 中，本身也是调用的  getMap 然后放进去 |
| get | 从 ThreadLocal 里面获取一个值 |
| ThreadLocalMap | 这个结构为了保存 ThreadLocal 对应的变量，里面的 Entry 继承弱引用，为了保存对应的 ThreadLocal 和 Value ，名字叫 Map 实际上里面核心的是一个 Entry 数组，一个 Thread 里面可以有多个 ThreadLocal 都是存在这个结构的 |
| getMap | 拿到对应的 ThreadLocalMap 然后根据本身 ThreadLocal 去取对应的值，这个的计算是利用位与完成的，找不到就会调用 setInitialValue |

列一下主要的方法：

```java
// getMap 这里可以看到，拿到的是 Thread 身上的 ThreadLocals 这个实际上就是一个 ThreadLocalMap
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

// set 与 get 核心都是围绕这 map 去搞的。
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

//  setInitialValue 与 initialValue
private T setInitialValue() {
        T value = initialValue();
}

// ThreadLocalMap 中 getEntry 核心实现
    private Entry getEntry(ThreadLocal<?> key) {
        // 字节码 & 找到目标位置     private final int threadLocalHashCode = nextHashCode(); 每个 ThreadLocal 都有自己的 HashCode
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            // 一个 Thread 可能有多个 ThreadLocal，找到了就返回i Entry 然后从 Entry 上面取下来
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

// Entry 狠简单就是为了存数据，别的木有啥好说的了，继承 WeakRefrence
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }        
```
