---
layout: post
title: 学习记录
subtitle: 记录一些暂时规模不大的学习记录，如果有必要可以扩展为单独成篇的文章
image: /img/lifedoc/jijiji.jpg
tags: [学习]
---

### 0718 学习了新版viewmodel存储数据

viewmodel的重建过程：

和前面相结合，一切都说得通了。我们再来捋一捋viewModel存储、恢复数据的过程。


1. 第一次创建ViewModel时，ViewModelStore利用HashMap将新创建的ViewModel对象存储了起来；
2. 在手机横竖屏时，Activity被销毁之前，会触发onRetainNonConfigurationInstance，对ViewModelStore进行存储；
3. Activity重建后，Activity会重新走onCreate生命周期，并且会再次去获取ViewModel对象。而这次ViewModel的获取与第一次创建不同，它会通过ViewModelStoreOwner先获取该Activity重建之前所保存的ViewModelStore，接着在ViewModelStore中根据Key，找到重建之前的ViewModel，进而恢复数据。

上述是大致流程，但是更详细的系统在哪里去存起来对应的 **Activity.NonConfigurationInstances lastNonConfigurationInstances** 实际上还没有找到，这里保存着viewmodel store的父数据结构，在这里叫activity，那么具体存起来的时候在哪里呢？看代码：

```java
/* 这个数据结构就是在ActivityThread里面定义的，而ActivityClientRecord里面保存了NonConfigurationInstances  
*/
final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();

/* 是哪里赋值的呢,在第一次创建Activity的时候就会放进去，如果此时但是此时我们的主角别置为空了，那么什么时候给置为非空呢，是在performDestroyActivity中，销毁之前让你保存状态，
*/
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    synchronized (mResourcesManager) {
                  mActivities.put(r.token, r); // 这里仅仅初始化r，占坑
    }
}

ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing, int configChanges, boolean getNonConfigInstance, String reason) {
if (getNonConfigInstance) {
                  try {
                      r.lastNonConfigurationInstances
                              = r.activity.retainNonConfigurationInstances();// 这个咱们ComponentActivity重写的，提供对应的缓存
                  } catch (Exception e) {
                  }
              }
}
```

到这里，viewmodel是怎么缓存的已经解开其神秘的面纱了，已经不是retainFragmet那一套，而仅仅是一些数据结构去保存，这名字也挺好，NonConfigurationInstances，不随着配置改变的东西，对于viewmodel来说，无论怎么转屏因为只是一份共有的数据 ，所以这里保存好之后，新的还是会接到咱们viewmodel里面的数据的，少了加载过程，省流量体验好。

最终这里虽然Activity销毁重建了，但是ActivityThread已经在onCreate前通过attach给到Activity这些东西了，我们想想为什么设计的在attach之后置空呢，这个是因为既然新的已经拿到viewmodel了，那么我们就要把缓存的干掉，因为不能保证新的不会改变viewmodel或者类似NoConfig里面的内容，这样能够始终保存last的状态，也是人家名字的含义，真棒。
