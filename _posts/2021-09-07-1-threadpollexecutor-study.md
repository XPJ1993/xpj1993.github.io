---
layout: post
title: Java 线程池学习
subtitle: 线程池是 Java 编程中非常好用的一个工具，对于多线程编程，这里简直是神器
image: /img/lifedoc/jijiji.jpg
tags: [ThreadPool]
---

### 基础用法

Java 线程池学习

**线程池创建方式：**

| 创建线程池 | 创建方式 |
|---|---|
| 通过 Executors | newFixedThreadPool(int count) 创建一个核心线程和最多线程数量相同的线程池 |
| 同上 | **newWorkStealingPool(int parallelism)** 创建一个支持窃取任务的线程池，https://www.cnblogs.com/shijiaqi1066/p/4631466.html 这个主要是多个线程有维护他们自己的任务队列，有的线程忙完了，会随机获取别的线程的任务去做 |
| 同上 | newSingleThreadExecutor() 创建一个单线程线程池 |
| 同上 | newCachedThreadPool 创建一个即时使用 1 分钟之后销毁多于线程的线程池，值得注意的这个线程池是没有核心线程的，一般处理瞬时任务量很大，干完这一票就没有的功能 |
| 同上 | newSingleThreadScheduledExecutor() 支持按照一定规则的创建 task 的线程池，可以 delay 一个 task |
| 同上 | newScheduledThreadPool(int size) 创建一个支持 delay task 的线程池，核心线程和最大线程数量一样 |
| new ThreadPoolExecutor() | 通过 new 基本的 ThreadPoolExecutor 去创建线程池，其中有几个参数需要知道他们的含义 |
| 创建自定义继承 ThreadPoolExecutor 的类 | 继承 ThreadPoolExecutor 通过这个可以控制更多的东西，例如 task 执行前和执行后都会有回调，我们可以在这个回调里面去做一些自定义的工作，开始加戏以及结束收尾 |

**线程池参数：**

| 线程池参数/方法 | 参数释义 |
|---|---|
| corePoolSize | 核心线程数 |
| maximumPoolSize | 最大线程数，用完之后会销毁释放资源的线程 |
| keepAliveTime | 非核心线程空闲多久释放线程 |
| TimeUnit |  时间单位 |
| BlockingQueue<Runnable> | 阻塞队列，当核心线程满载就会往这个队列添加，满了之后再开启非核心线程，最后在满了就会触发拒绝策略 |
| ThreadFactory | 线程构造工厂接口类，负责生成对应的线程 |
| RejectedExecutionHandler |  拒绝策略执行触发类，这里可以自定义实现在拒绝的时候再把这个 task （Runnable） 添加回去构造一个不丢任务的线程池，并且队列长度有限不浪费资源 |
| <span style="color:#871F78;">beforeExecute</span> | 参数为 Thread t，Runnable r 运行一个 task 的时候提前通知，**这个是方便在执行一个 task 的时候给这个 task 增加一些戏，例如装饰器加东西** |
| <span style="color:#871F78;">afterExecute</span> | 参数为 Thread t，Runnable r 运行一个 task 的时候运行完之后的后置提醒**这个是在执行完成后方便借助这个做一个 task 收尾的工作** |
| terminated | 线程池终止之后的回调**整体线程池的终止回调，可以通过这个去回收共用的工具或者连接池等等** |

### 线程池原理

线程池的原理就是 worker 的不断 runWorker 然后他们共享的去 BlockingQueue 里面取还没有消费的 task，另外，值得一提的是这玩意里面最神奇的是哪个 ctl ，大家对这个左移右移啥的各种骚操作给他赋予了很多的含义，其中有当前 worker count， 有 是否已经 SHUTDOWN 等等，使用位运算少了很多变量，并且利用按位与或者按位或可以提高运行效率，毕竟这玩意计算机看得懂。

```java
 public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        // 1 worker 数量小于核心线程
        if (workerCountOf(c) < corePoolSize) {
            // 尝试添加，成功就返回，否则继续
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 判断还没有挂壁那么就往阻塞队列塞数据
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再次 check 一边，没有运行且移除掉这个 task 之后 触发 reject
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0) // 如果此时 worker 为 0 了，创建一个新的 worker， 这里 task 为 null 也是允许的只是说没有第一个任务而已，因为上边已经加入任务到队列了，这里在 runWorker 的时候还会去队列里面取数据，这里如果不为空会重复执行 task
                addWorker(null, false);
        }
        // 最终把这个 task 添加到非核心线程中，这里如果 worker count 大于等于 maximumPoolSize 就会触发拒绝
        else if (!addWorker(command, false))
            reject(command);
    }


    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                // 我把 BlockingQueue 的 size 设置的比较小的时候，这里极容易发生 reject 原因是 wc 大于等于 maximumPoolSize
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 这里的 break retry 就直接不会进到 for 循环中，c 经过 cas 之后 +1 成功就会推出这个双重循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                // 这里是共享一把可重入锁，因为要操作 workers
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 这里就算添加 worker 到列表了
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 这里启动 worker 中的 thread
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // add 失败会减一并且尝试清除任务，并且触发终止线程池操作，很多信号会调用 tryTerminate ，他的名字是 try 不一定立刻终止，因为有任务在的话还是会保证任务执行完毕的
            if (! workerStarted)
                addWorkerFailed(w);
        }
        // 添加成功返回 true
        return workerStarted;
    }


    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        // 可空的第一个 task ，这里为空在 runWorker 的时候会从 getTask 获取
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        // 保存每个 worker 完成的 task 数量
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            // 这里自己先加锁
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker. */
        public void run() {
            // 这里会释放锁先，为了可以被打断
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

    final void runWorker(Worker w) {
        // 这里为啥取当前 Thread 而不是拿 worker 的 Thread 呢，是因为调用到这里的时候必然在 worker 里面的 thread 执行的，因此这里和拿 worker thread 效果一样
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 这里 task 如果为空，那么就通过 getTask 去从队列里面取
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行拿到的 task 
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    // worker 的执行成功加一
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 对 worker 的收尾
            processWorkerExit(w, completedAbruptly);
        }
    }

```

### 线程池使用场景

使用场景还蛮多的，例如 okhttp 的就是各种线程池的使用，glide 也是使用线程池去缓存啥的。
一般我们用的时候需要自定义的时候可以继承然后在相应回调里面加戏。

### 线程池总结

线程池很强大，这在于 Java 大师们优雅的实现，从中可以学到多用 位运算，一个 ctl 通过各种位运算代表了那么多的含义，核心是一个 AtomicInteger 去管控这些，目前自己还达不到这些 level ，但是良好的使用线程池还是能够做的了的。
另外，线程池贴心设计的 before after 好使呀。
