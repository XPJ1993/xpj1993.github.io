---
layout: post
title: Leakcanary 解析
subtitle: 内存泄漏监控工具解析
image: /img/lifedoc/jijiji.jpg
tags: [三方库]
---

三方库总目录：
[三方库](https://xpj1993.github.io/2021-09-10-02-third-libary/)

### Leakcanary 解析

按照解析第一个 Arouter 的套路来，首先是使用，然后想如果自己做怎么做，然后分析别人是怎样实现这个功能的。

| 文章目录 | 代码解析 |
|---|---|
| 官方解读 | https://square.github.io/leakcanary/fundamentals-how-leakcanary-works/ |
| 使用方式 | 可太简单了，直接在使用的 app build 文件下添加 debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.7' 即可，leak 会自动的初始化开始工作，太强大了吧 |
| 核心原理 | 持有弱引用队列，如果在 onDestory 5s 之后这里没有被触发释放，那么就认为他内存泄漏了，这时候就会在 leakcanary 自己的线程中开始 dump hprof 文件，这个文件为我们查看更多的内存信息提供依据 |
| 内存泄漏的几种场景 | 匿名内部类、非静态 handler、单例持有 activity 引用、监听器注册之后没有反注册、匿名 thread Runnable asyntask 、容器中内存不释放（会导致内存吃紧） |
| 内存抖动 | 内存抖动的场景，大对象的频繁创建于释放，解决方案，对象池 |

#### Leakcanary 使用

```groovy
// 这就完事了
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.7'
```

#### 初始化解析

首先在 leakcanary-object-watcher-android 中有 AppWatcherInstaller（他是个密封类，最终实际上调用是他的继承者 MainProcess ） 类，这个类继承了 ContentProvider 类，这样就会在启动 App 的时候触发他具体代码如：

```kotlin
// 调用 content provider 的 onCreate 方法时会调用 AppWatcher manualInstall 方法完成初始化
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }

// 初始化代码，默认 5s 超时，默认的 watchers 为 构建包含四种的 watcher
  fun manualInstall(
    application: Application,
    retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
    watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)
  ) {
      // 1. 主线程判断
    checkMainThread()
    if (isInstalled) {
      throw IllegalStateException(
        "AppWatcher already installed, see exception cause for prior install call", installCause
      )
    }
    // 2. 设置的 delay 超时提醒时间需要大于 0
    check(retainedDelayMillis >= 0) {
      "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
    }
    installCause = RuntimeException("manualInstall() first called here")
    this.retainedDelayMillis = retainedDelayMillis
    // 3. debug 版本启动 LogcatShark
    if (application.isDebuggableBuild) {
      LogcatSharkLog.install()
    }
    // 4. 初始化 InternalLeakCanary
    /*
    InternalLeakCanary#invoke
    此回调中会初始化一些检测内存泄露过程中需要的对象：
    */
    // Requires AppWatcher.objectWatcher to be set
    LeakCanaryDelegate.loadLeakCanary(application)

    // 5. 通知几个监听器去 install 自己
    watchersToInstall.forEach {
      it.install()
    }
  }
```

几个 install：

```kotlin
// activity 
  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

// fragment and viewmodel
private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityCreated(
        activity: Activity,
        savedInstanceState: Bundle?
      ) {
        for (watcher in fragmentDestroyWatchers) {
          watcher(activity)
        }
      }
    }

  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }  


  companion object {
    private const val ANDROIDX_FRAGMENT_CLASS_NAME = "androidx.fragment.app.Fragment"
    // 这里面都会针对 viewmodel 的 onclear 进行监听
    private const val ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME =
      "leakcanary.internal.AndroidXFragmentDestroyWatcher"

    // Using a string builder to prevent Jetifier from changing this string to Android X Fragment
    @Suppress("VariableNaming", "PropertyName")
    private val ANDROID_SUPPORT_FRAGMENT_CLASS_NAME =
      StringBuilder("android.").append("support.v4.app.Fragment")
        .toString()
    private const val ANDROID_SUPPORT_FRAGMENT_DESTROY_WATCHER_CLASS_NAME =
      "leakcanary.internal.AndroidSupportFragmentDestroyWatcher"
  }  

// RootViewWatcher 的 install， 
  override fun install() {
    Curtains.onRootViewsChangedListeners += listener
  }

// RootViewSpy
RootViewsSpy.install()

return RootViewsSpy().apply {
        WindowManagerSpy.swapWindowManagerGlobalMViews { mViews ->
          delegatingViewList.apply { addAll(mViews) }
        }
      }

// WindowManagerSpy 功能
// Enables replacing WindowManagerGlobal.mViews with a custom ArrayList implementation.

// 反射拿到这个对象，然后去注册监听
  val className = if (SDK_INT > 16) {
      "android.view.WindowManagerGlobal"
    } else {
      "android.view.WindowManagerImpl"
    }

// service watcher 的 install
  override fun install() {
    checkMainThread()
    check(uninstallActivityThreadHandlerCallback == null) {
      "ServiceWatcher already installed"
    }
    check(uninstallActivityManager == null) {
      "ServiceWatcher already installed"
    }
    try {
      swapActivityThreadHandlerCallback { mCallback ->
        uninstallActivityThreadHandlerCallback = {
          swapActivityThreadHandlerCallback {
            mCallback
          }
        }
        Handler.Callback { msg ->
          if (msg.what == STOP_SERVICE) {
            val key = msg.obj as IBinder
            activityThreadServices[key]?.let {
              onServicePreDestroy(key, it)
            }
          }
          mCallback?.handleMessage(msg) ?: false
        }
      }
      swapActivityManager { activityManagerInterface, activityManagerInstance ->
        uninstallActivityManager = {
          swapActivityManager { _, _ ->
            activityManagerInstance
          }
        }
        Proxy.newProxyInstance(
          activityManagerInterface.classLoader, arrayOf(activityManagerInterface)
        ) { _, method, args ->
          if (METHOD_SERVICE_DONE_EXECUTING == method.name) {
            val token = args!![0] as IBinder
            if (servicesToBeDestroyed.containsKey(token)) {
              onServiceDestroyed(token)
            }
          }
          try {
            if (args == null) {
              method.invoke(activityManagerInstance)
            } else {
              method.invoke(activityManagerInstance, *args)
            }
          } catch (invocationException: InvocationTargetException) {
            throw invocationException.targetException
          }
        }
      }
    } catch (ignored: Throwable) {
      SharkLog.d(ignored) { "Could not watch destroyed services" }
    }
  }

```


一些 检测时机：

```java
ActivityWatcher
Activity#onActivityDestroyed
FragmentAndViewModelWatcher
Fragments (Support Library, Android X and AOSP)
Fragment#onDestroy()
Fragment views (Support Library, Android X and AOSP)
Fragment#onDestroyView()
Android X view models (both activity and fragment view models)
ViewModel#onCleared()
Expects root views to become weakly reachable soon after they are removed from the window

RootViewWatcher
rootView.addOnAttachStateChangeListener#onViewDetachedFromWindow
ServiceWatcher
AMS#serviceDoneExecuting
```

#### 核心实现解析

ObjectWatcher

// 0. 从各个源头调用 expectWeaklyReachable 方法
![](https://raw.githubusercontent.com/XPJ1993/images/master/20210914110607.png)

```kotlin

  // 存起来如果泄露要通知的监听者
  private val onObjectRetainedListeners = mutableSetOf<OnObjectRetainedListener>()

  /**
   * References passed to [watch].
   */
   // 核心数据结构，这里存起来我们要监听的 key 弱引用
  private val watchedObjects = mutableMapOf<String, KeyedWeakReference>()

  // 弱引用队列
  private val queue = ReferenceQueue<Any>()

// 1. 将被监听的对象使用 弱引用 持有之
  @Synchronized override fun expectWeaklyReachable(
    watchedObject: Any,
    description: String
  ) {
    if (!isEnabled()) {
      return
    }
    // 2. 首先从 queue 里面搞出去一个弱引用队头的 对象
    removeWeaklyReachableObjects()
    /*
       3. key / watchTime / refrence
    */
    val key = UUID.randomUUID()
      .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    /*
      这里构建是咱们的核心 
      class KeyedWeakReference(
        referent: Any,
        val key: String,
        val description: String,
        val watchUptimeMillis: Long,
        referenceQueue: ReferenceQueue<Any>
      ) : WeakReference<Any>(
        referent, referenceQueue
      ) 
     这里可以看到这个 keyed 是继承于 WeakRefreence 的，继承的时候将，reference queue 设置为我们这个了，所以 在 Java GC 回收的时候才会把这个放到 queue 里，这样我们才有了依据    
    */
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
      "Watching " +
        (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
        (if (description.isNotEmpty()) " ($description)" else "") +
        " with key $key"
    }

    // 4. 添加刚刚构建的 refrence 到 map
    watchedObjects[key] = reference
    // 5. executor 里去 exec 去
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
  }

// 5.1 看看这个 checkRetainedExecutor 是个啥，lambda 表达式，delay 5s 之后检测是否还在
checkRetainedExecutor = {
      check(isInstalled) {
        "AppWatcher not installed"
      }
      mainHandler.postDelayed(it, retainedDelayMillis)
    }

// 如果超时了这个对象还没有被移除，那么就是超时了
  @Synchronized private fun moveToRetained(key: String) {
    // 这里去尝试移除对应的对象，这里移除成功之后理论上 watchedObjects 这里就木有了，有就代表泄漏，妙啊，妙啊。  
    removeWeaklyReachableObjects()
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
  }

// 尝试移除已经解除嫌疑滴对象，因为这里到达就说明正常释放了
  private fun removeWeaklyReachableObjects() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
    do {
      ref = queue.poll() as KeyedWeakReference?
      if (ref != null) {
        watchedObjects.remove(ref.key)
      }
    } while (ref != null)
  }  
```

#### 昨天的车上碎念

今天工作时间一天就产出了一篇文章，这是不是效率也太低了，主要是这篇文章也是之前学习过的 arouter 而写的。

继续要写的是 leakcanary 的解析，这个希望明天上午可以写完，现在已经写到核心类 object wacther 这里了，下一步就是解析他的原理，实际上，这个里面就是存储着键值对，其中 value 为各个监控对象的 弱引用 将他们放到弱引用队列等各种周期的触发，例如 activity 的 ondestory ， fragment 的 ondestory 及 ondestoryview 以及 viewmodel 的 onclear ，service 的 ams destory service，view 的 ondetachview 等时机，这个主要是是反射了 windowrootimpl 等，将监听加进去。

收到触发信号后就开始计时，有一个默认的超时时间，就是 5s ，5s 后这个弱引用没有被添加到引用队列的话，那么就判定为泄露了，这时候就会通知各种监听的 listener，他们去做 dump 了，更新 notification 了，等等。

这个分析完之后分析 blockcanary 这个主要是在 looper 里埋地雷，超时就会 dump 数据然后会统计超时了多少，还有一个功能是 watchdog 就是一个 while true 循环，每当有事件发生时，他就搞一个 int 的 task 到 looper 里，超时之后发现这个值没有变化的话就证明这个没有执行到，产生 anr 了，然后统计当前内存和线程信息，分析 anr 产生的原因。

其实细数上面两个 canary ，其中 leak 属于开创，用了各种生命周期的时机去触发，然后利用 Java 虚拟机的内存回收机制，就是回收的对象一定会先放到 弱引用 队列中去，结合这两者就能巧妙的解决内存泄露的问题了。

内存抖动问题可以通过 as 的 profiler 去观察，看着内存一直有 波峰波谷 的震动，那么就证明有内存波动的存在，以及虽然没有内存泄露，但是有 collection 大量持有大内存不释放会引起 oom，通过分析 hprof 也可以查看那些占用内存最大，然后想办法干掉或者减少这些内存大户的占用，最主要的就是图片内存的占用了。

#### todo 

接下来分析的三方库：

glide

retrofit

okhttp

hilt

gson

eventbus

room

### leakcanary 总结

leak canary 的核心就是<span style="color:#871F78;">在各个关键地方去巧妙的找到可能内存泄漏的点，找到之后让各个 watcher 去在这些时机将这些对象都放到一个 map 中</span>，其中 key 为 uuid 搞得一个任意值，value 为 一个弱引用（如果是强引用，他都会导致内存泄漏。），然后<span style="color:#871F78;">利用 Java 的 GC 机制，一个对象在回收的时候会把它塞到 弱引用队列中去</span>，这时候咱们在一个 超时间隔之后去 check 他如果在 queue 里面，那么就把它移除，然后移除 map 中的存储，校验这个对象在 map 中没有了，证明他已经真正的回收了，否则就是泄漏了，完美。
