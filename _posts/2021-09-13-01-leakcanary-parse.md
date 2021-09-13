---
layout: post
title: Leakcanary 解析
subtitle: 内存泄漏监控工具解析
image: /img/lifedoc/jijiji.jpg
tags: [三方库]
---

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



