---
layout:     post  
title:      "LeakCanary 深入探索"  
subtitle:   "帮你检测内存泄漏"  
date:       2021-05-20 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - 性能优化 
    - 三方库 

---


## 参考

官网：https://square.github.io/leakcanary/

Github：https://github.com/square/leakcanary

LeakCanary原理从0到1：https://juejin.cn/post/6936452012058312740

[从 LeakCanary 探究线上内存泄漏检测方案](https://juejin.cn/post/6949759784253915172/)

JsonChao：[Android主流三方库源码分析（六、深入理解Leakcanary源码）](https://juejin.cn/post/6844904070977749005#heading-20)，这篇写得更详细。


## 问题

问题就是看源码时让你不会迷路和懵逼的方向标。

LeakCanary 的功能概括：检测泄漏并发送提示到通知栏，点击跳转可查看泄漏到路径。

怎么检测泄漏？
检测到泄漏之后，怎么拿到泄漏路径？
怎么发通知到通知栏？

## 源码开始

初始化代码：

```
    LeakCanary.install(this)
```

### install

```
  public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application)
    .listenerServiceClass(DisplayLeakService.class)
    .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
    .buildAndInstall();
  }

// 主要看 buildAndInstall 函数
```

### buildAndInstall

```
  public @NonNull RefWatcher buildAndInstall() {
    // 第二遍会报错
    if (LeakCanaryInternals.installedRefWatcher != null) {
      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    // refWatcher 对象，用于监听 Activity 和 Fragment
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
      if (watchActivities) {
        // 1，监听 Activity，传入 refWatcher
        ActivityRefWatcher.install(context, refWatcher);
      }
      if (watchFragments) {
        // 2，监听 Fragment，传入 refWatcher
        FragmentRefWatcher.Helper.install(context, refWatcher);
      }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    // 返回 refWatcher 对象
    return refWatcher;
  }
```

注释1、2，可知监听 Activity 或 Fragment 都适用 refWatcher 对象。
选择继续跟踪注释1的源码：

### ActivityRefWatcher#install

```
// com/squareup/leakcanary/ActivityRefWatcher.java

  public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
    // 注册所有 Activity 的生命周期回调
    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
  }
  
```

`application.registerActivityLifecycleCallbacks()`
 这个注册，可以收到所有生命周期函数的回调。因此可以监听到每个 Activity 的创建和销毁。

那么去看看接收回调的 ActivityRefWatcher 都做了什么？

### onActivityDestroyed

```
// com/squareup/leakcanary/ActivityRefWatcher.java

  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityDestroyed(Activity activity) {
          refWatcher.watch(activity);
        }
      };
```

### RefWatcher#watch

当 Activity/Fragment 被销毁后，refWatcher 开始 watch 函数；

找到 refWatcher 的真正实现类：

```
// com/squareup/leakcanary/RefWatcher.java

  public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }
  
  /**
   * Watches the provided references and checks if it can be GCed. This method is non blocking,
   * the check is done on the {@link WatchExecutor} this {@link RefWatcher} has been constructed
   * with.
   *
   * @param referenceName An logical identifier for the watched object.
   */
  public void watch(Object watchedReference, String referenceName) {
    // watchedReference 就是 Activity
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    ensureGoneAsync(watchStartNanoTime, reference);
  }
```
观察 Activity destroy ，是通过 Application 注册生命周期函数做到的。

当 Activity destroy ，会触发 refWatch 的 watch 函数，开始 watch 这个 activity。

![IMG_15](/img/media/16214134386865/IMG_15.jpg)


### RefWatcher#ensureGoneSync

开始 watch 时，ensureGoneSync 函数会让 watchExecutor 接收一个 runnable。

![IMG_16](/img/media/16214103204099/IMG_16.jpg)


watchExecutor 的 excute 函数，会扔一个任务到 IdleHander 中，当主线程空闲，就会执行 runnable 的 run 函数！

```
// com/squareup/leakcanary/AndroidWatchExecutor.java

  @Override public void execute(@NonNull Retryable retryable) {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      waitForIdle(retryable, 0);
    } else {
      postWaitForIdle(retryable, 0);
    }
  }
```

主线程中执行：waitForIdle(retryable, 0);

```
    private void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }
```

主线程空闲时，执行该函数 postToBackgroundWithDelay ：

```
  private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    // 在主线程中往子线程 post 一个任务
    backgroundHandler.postDelayed(new Runnable() {
      @Override public void run() {
        // 最终就是执行这一句
        // 1
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
  }
```

注释1中的 retryable 就是 ensureGoneSync 函数中中的 Retryable 对象，在主线程空闲时，它的 run 函数在子线程中执行！

![IMG_14](/img/media/16214103204099/IMG_14.jpg)

之所以要等主线程空闲，再去子线程执行任务，就是因为主线程空闲，说明当时没有操作了，
- 一来进行垃圾回收引起的卡顿影响最小；
- 二来可能是用户退出某个了 Activity 。

### 泄漏判断的核心逻辑

回头看看 watch 函数做了什么：

![IMG_17](/img/media/16214103204099/IMG_17.jpg)

#### 判断泄漏的核心逻辑

在 destroy 一个 Activity 时，
创建一个随机数作为 key ，关联 Activity，
创建一个弱引用关联Activity 、key 和 queue，
同时 key 存到 retainedKeys 中（又称怀疑名单）。
当主线程空闲时，会在子线程开始「排查怀疑名单」：
> 循环从 queue 中取出弱引用，
> 每取到一个弱引用，证明其关联的 Activity 被 GC 回收，所以 key 也可以从怀疑名单中移除。
> 这样直到 queue 为空，再看 key 集合中剩余的 key，其对应的 Activity 还没有被回收！否则其弱引用就会出现在 queue 中！
> 通过该逻辑，可检测出泄露的 Activity。

再继续看 ensureGone 函数：

### RefWatcher#ensureGone

![IMG_18](/img/media/16214103204099/IMG_18.jpg)

### removeWeaklyReachableReferences

还是 RefWatcher 类的函数：

![IMG_19](/img/media/16214103204099/IMG_19.jpg)


**下面代码，整好印证了上面的说法！**
注意，这里会有一个优化处理：在没有主动 GC 时，先清理一次怀疑名单；
这么做的好处是：万一能把 Activity 从怀疑名单排除，就免去了主动GC的需要。
达到优化性能的目的。

回头看 ensureGone 函数：

![IMG_20](/img/media/16214103204099/IMG_20.jpg)

##### 怎么正确的手动调用 GC

上述的 `gcTrigger.runGc` 代码，触发了 gc。看起源码：

```
public interface GcTrigger {
  GcTrigger DEFAULT = new GcTrigger() {
    @Override public void runGc() {
      // Code taken from AOSP FinalizationTest:
      // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
      // java/lang/ref/FinalizationTester.java
      // System.gc() does not garbage collect every time. Runtime.gc() is
      // more likely to perform a gc.
      Runtime.getRuntime().gc();
      enqueueReferences();
      System.runFinalization();
    }

    private void enqueueReferences() {
      // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
      // references to the appropriate queues.
      try {
        Thread.sleep(100);
      } catch (InterruptedException e) {
        throw new AssertionError();
      }
    }
  };

  void runGc();
}
```

上述的 DEFAULT 对象，实现了 runGc 方法，从其注释可知，System.gc() 并不总是能触发，因而采用 Runtime.gc()，并休眠 100 ms。这就是正确的触发 gc 的方法。

### 发现泄漏小结

上述的 ensureGone 函数中，如果 gone(reference) 返回 false，则存在泄漏。
到此发现泄漏的逻辑走完了，应该有小结：

小结其实可以参考 「判断泄漏的核心逻辑」标题下的总结。。。偷懒了哈哈～～～

接着看看怎么分析泄漏。。。

### 分析泄漏开始

从 RefWatcher#ensureGone 函数开始看：

```
  @SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
  
    // 省了判断泄漏的代码...
    
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      // 1，
      // 创建 dump 文件，并写入 dump 数据，调用了 SDK 自带的 Debug 类的函数
      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);

      // 2，
      // 把 dump 和此次销毁的 Activity 或 Fragment相关的 key、reference 等信息，
      // 封装到一个 HeapDump 对象
      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();

      // 3，
      // 分析 dump 信息中是否有泄漏
      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }
```

看图解析：

![屏幕快照 2021-04-21 下午4.26.35](/img/media/16214103204099/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-04-21%20%E4%B8%8B%E5%8D%884.26.35.png)

接下来讲一些细节，怎么获取到堆信息的。

从注释1中的 `File heapDumpFile = heapDumper.dumpHeap();` 函数点击去看：
找到这个类：**AndroidHeapDumper.java** 的 dumpHeap() 函数：
![IMG_23](/img/media/16214103204099/IMG_23.jpg)


再点击 `Debug.dumpHprofData(heapDumpFile.getAbsolutePath());` 进去，

```
    /**
     * Dump "hprof" data to the specified file.  This may cause a GC.
     *
     * @param fileName Full pathname of output file (e.g. "/sdcard/dump.hprof").
     * @throws UnsupportedOperationException if the VM was built without
     *         HPROF support.
     * @throws IOException if an error occurs while opening or writing files.
     */
    public static void dumpHprofData(String fileName) throws IOException {
        VMDebug.dumpHprofData(fileName);
        // 不能往下追了！
    }
```

Debug 是 SDK 自带的类，
**我们也可以通过 Debug 的 dumpHprofData 方法，拿到 dump 的信息！**

在拿到了 dump 信息后，跟踪注释3处，到这里去分析 dump 信息：
headDumpListener 对象的 analysis 函数；

### ServiceHeapDumpListener#analysis

![IMG_22](/img/media/16214103204099/IMG_22.jpg)

headDumpListener 对象的创建，是在 install 函数中的 listenerServiceClass(DisplayLeakService.class) 调用中：


```
public @NonNull AndroidRefWatcherBuilder listenerServiceClass(
  @NonNull Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
}
```


```
public final class ServiceHeapDumpListener implements HeapDump.Listener {
    ...    
    public ServiceHeapDumpListener(@NonNull final Context context,
        @NonNull final Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
      this.listenerServiceClass = checkNotNull(listenerServiceClass, "listenerServiceClass");
      this.context = checkNotNull(context, "context").getApplicationContext();
    } 
    ...
}
```

在 ServiceHeapDumpListener 中保存了一个 DisplayLeakService 对象和 application对象。ServiceHeapDumpListener 的作用就是接收 一个 dump 文件去分析。
在 ServiceHeapDumpListener 的 analysis 函数中：
调用 HeapAnalyzerService 的 runAnalysis 函数，
启动 HeapAnalyzerService 这个前台服务：

### HeapAnalyzerService#runAnalysis

```
  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabledBlocking(context, HeapAnalyzerService.class, true);
    setEnabledBlocking(context, listenerServiceClass, true);
    // 传入自身类型，自动自己
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    // 传入 heapDump，将在服务中的子线程中分析 dump 信息
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    ContextCompat.startForegroundService(context, intent);
  }
```

再次回顾调用链：
-> Activity | Fragment 销毁 
-> watch 
-> ensureGoneAsync
-> 放一个 retryAble，主线程空闲时，在子线程执行 retryAble
-> ensureGone 函数
-> heapdumpListener.analyze(heapDump)
-> HeapAnalyzerService#runAnalysis

由此可知，runAnalysis 该函数是在主进程的子线程中执行的，
startForegroundService 函数会新建一个前台服务，
该前台服务，就是 HeapAnalyzerService，是在独立进程的子线程中运行；
onHandleIntentInForeground 方法，也是在独立进程的子线程中运行。

### HeapAnalyzerService 独立进程

HeapAnalyzerService 是运行在一个独立进程中的：

```
// ../leakcanary-android-1.6.3/AndroidManifest.xml

    <application>
        <service
            android:name="com.squareup.leakcanary.internal.HeapAnalyzerService"
            android:enabled="false"
            android:process=":leakcanary" />
    ...
```

HeapAnalyzerService 的继承关系：
> HeapAnalyzerService
> extends ForegroundService 
> extends IntentService

IntentService 的特性是，在 onCreate 的时候，创建一个带有子线程的 handler，并在 onStart 的时候，把 intent 放入 msg 中发送给 handler 处理，该 handler 会最终调用 onHandleIntent 函数来处理 intent；

ForegroundService 实现了 onHandleIntent 函数，调用 onHandleIntentInForeground 函数来处理 intent；

问题插入：
> 为什么要多调用这个 onHandleIntentInForeground 函数呢？
> 直接在 onHandleIntent 函数处理 intent 不行吗？
> ForegroundService 做了啥？

知识点插入：Android8.0 之后，不允许直接调用 startService；
参考：[Android 8.0启动后台服务](https://blog.csdn.net/henkun/article/details/88867067)
> Android8.0以后系统不允许直接采用startservice的方式启动后台服务，官网描述大概意思是这样儿式的：

> - 如果针对 Android 8.0 的应用尝试在不允许其创建后台服务的情况下使用 startService()函数，则该函数将引发一个 IllegalStateException。
> - 新的 Context.startForegroundService() 函数将启动一个前台服务。现在，即使应用在后台运行，系统也允许其调用 Context.startForegroundService()。不过，应用必须在创建服务后的五秒内调用该服务的 startForeground()函数。

也就是说，Android 8.0 后，想要启动服务，就得调用这两个函数：
- Context.startForegroundService()
- startForeground()

ForegroundService 类的工作，就是先调用了 startForeground() 函数；
其子类再调用 Context.startForegroundService() 函数启动服务即可；

ForegroundService 类的源码：

![屏幕快照 2021-04-21 下午3.08.48](/img/media/16214103204099/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-04-21%20%E4%B8%8B%E5%8D%883.08.48.png)

到此再回顾 HeapAnalyzerService#runAnalysis 函数：

```
  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabledBlocking(context, HeapAnalyzerService.class, true);
    setEnabledBlocking(context, listenerServiceClass, true);
    // 传入自身类型，自动自己
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    // 传入 heapDump，将在服务中的子线程中分析 dump 信息
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    // 1
    ContextCompat.startForegroundService(context, intent);
  }
```

可知注释1，startForegroundService 函数传递的 intent，会在 onHandleIntentInForeground 函数中处理，onHandleIntentInForeground 函数在独立进程的子线程中运行；

### HeapAnalyzerService#onHandleIntentInForeground

```
  @Override protected void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    // 1，新建一个分析对象
    HeapAnalyzer heapAnalyzer =
        new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);

    // 2，检查泄漏，并返回结果
    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
        heapDump.computeRetainedHeapSize);
        
    // 3，发送结果
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
```

跟着注释2，进入源码：

### HeapAnalyzer#checkForLeak

```
  public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile,
      @NonNull String referenceKey,
      boolean computeRetainedSize) {
    long analysisStartNanoTime = System.nanoTime();

    ...

    try {
      // 这里用到了 haha 库的组件来解析 dump 文件
      HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
      HprofParser parser = new HprofParser(buffer);
      // 解析得到一个快照
      Snapshot snapshot = parser.parse();
      // 根据 referenceKey 在快照中寻找 Activity
      Instance leakingRef = findLeakingReference(referenceKey, snapshot);

      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        String className = leakingRef.getClassObj().getClassName();
        return noLeak(className, since(analysisStartNanoTime));
      }
      // 到这说明找到了 Activity 的泄漏，由 leakingRef 字段引用
      // 1，调用 findLeakTrace 并返回其返回值
      return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
    } catch (Throwable e) {
      return failure(e, since(analysisStartNanoTime));
    }
  }
```

该 checkForLeak 函数中，调用了 haha 库解析 dump 文件，返回 leakingRef 变量代表泄漏的对象；
如果 leakingRef 不为空，则表示找到了泄漏对象，继续下一步构建泄漏路径；
跟着注释1，进入源码：

#### HeapAnalyzer#findLeakTrace

```
  private AnalysisResult findLeakTrace(long analysisStartNanoTime, Snapshot snapshot,
      Instance leakingRef, boolean computeRetainedSize) {

    listener.onProgressUpdate(FINDING_SHORTEST_PATH);
    
    // 1，路径 finder
    ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
    ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef);

    String className = leakingRef.getClassObj().getClassName();

    // False alarm, no strong reference path to GC Roots.
    // 没有到 GC Roots 的强引用
    if (result.leakingNode == null) {
      return noLeak(className, since(analysisStartNanoTime));
    }

    listener.onProgressUpdate(BUILDING_LEAK_TRACE);
    // 2，从泄漏到节点，构建路径
    LeakTrace leakTrace = buildLeakTrace(result.leakingNode);

    ...

    // 3，返回结果
    return leakDetected(result.excludingKnownLeaks, className, leakTrace, retainedSize,
        since(analysisStartNanoTime));
  }
```

该 findLeakTrace 函数中，
注释1处，调用 ShortestPathFinder 的 findPath 函数，其作用就是对每个GcRoot的引用链的堆结构进行BFS遍历，然后将泄漏实例所在节点包装在一个 Result 中并返回。此解析参考：[从 LeakCanary 探究线上内存泄漏检测方案](https://juejin.cn/post/6949759784253915172/)；
注释2处，调用了 buildLeakTrace 函数创建最短路径引用链，得到一个包含 泄漏路径的 leakTrace 对象；
注释3处，调用 leakDetected 函数，把 leakTrace 对象和其他信息封装起来，作为最终的结果返回。

下面看看如何构建泄漏路径：

跟着注释1，看看构建 buildLeakTrace 的逻辑：

#### HeapAnalyzer#buildLeakTrace

```
  private LeakTrace buildLeakTrace(LeakNode leakingNode) {
    List<LeakTraceElement> elements = new ArrayList<>();
    // We iterate from the leak to the GC root
    LeakNode node = new LeakNode(null, null, leakingNode, null);
    while (node != null) {
      LeakTraceElement element = buildLeakElement(node);
      if (element != null) {
        elements.add(0, element);
      }
      node = node.parent;
    }

    List<Reachability> expectedReachability =
        computeExpectedReachability(elements);

    return new LeakTrace(elements, expectedReachability);
  }
```

逻辑就是：从 leakingNode 这个节点开始，循环的向上找父节点，记录到 List<LeakTraceElement> elements 中；
最后返回一个 LeakTrace 对象，带有泄漏的路径节点；

回到上一级，在 findLeakTrace 函数的最后调用 leakDetected 函数，把 leakTrace 对象和其他信息封装起来，得到一个 AnalysisResult 对象，并返回；
跟着上上一段源码注释2，进入源码：

#### HeapAnalyzer#leakDetected

```
  public static @NonNull AnalysisResult leakDetected(boolean excludedLeak,
      @NonNull String className,
      @NonNull LeakTrace leakTrace, long retainedHeapSize, long analysisDurationMs) {
    return new AnalysisResult(true, excludedLeak, className, leakTrace, null, retainedHeapSize,
        analysisDurationMs);
  }
```

封装并返回一个 AnalysisResult 对象。

### 分析泄漏小结

分析泄漏的过程，主要都在 HeapAnalyzer#checkForLeak 函数中，都做了这些事：
- 生成 dump 文件，即 hprof 文件
- 调用 haha 库，解析 dump 文件，并拿到泄漏节点 Instance 类型的对象 leakingRef
- 根据泄漏节点，生成一个包含泄漏路径的 LeakTrace 对象 
- 把泄漏路径和其他相关信息，封装成 AnalysisResult 对象返回
- AnalysisResult 对象将在发送泄漏通知的时候使用

#### checkForLeak 函数解析参考

[Android主流三方库源码分析（六、深入理解Leakcanary源码）](https://juejin.cn/post/6844904070977749005#heading-20) 该文对 checkForLeak 方法的解析：
调用链：
HeapAnalyzerService#runAnalysis
HeapAnalyzerService#onHandleIntentInForeground
HeapAnalyzer#checkForLeak

> ```
> public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile,
>   @NonNull String referenceKey,
>   boolean computeRetainedSize) {
>     ...
>     
>     try {
>     listener.onProgressUpdate(READING_HEAP_DUMP_FILE);
>     // 1
>     HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
>   
>     // 2
>     HprofParser parser = new HprofParser(buffer);
>     listener.onProgressUpdate(PARSING_HEAP_DUMP);
>     Snapshot snapshot = parser.parse();
>   
>     listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
>     // 3
>     deduplicateGcRoots(snapshot);
>     listener.onProgressUpdate(FINDING_LEAKING_REF);
>   
>     // 4
>     Instance leakingRef = findLeakingReference(referenceKey, snapshot);
> 
>     // 5
>     if (leakingRef == null) {
>         return noLeak(since(analysisStartNanoTime));
>     }
>   
>     // 6
>     return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
>     } catch (Throwable e) {
>     return failure(e, since(analysisStartNanoTime));
>     }
> }
> ```
> 
> 在注释1处，会新建一个内存映射缓存文件buffer。
> 在注释2处，会使用buffer新建一个HprofParser解析器去解析出对应的引用内存快照文件snapshot。
> 在注释3处，为了减少在Android 6.0版本中重复GCRoots带来的内存压力的影响，使用deduplicateGcRoots()删除了gcRoots中重复的根对象RootObj。
> 在注释4处，调用了findLeakingReference()方法将传入的referenceKey和snapshot对象里面所有类实例的字段值对应的keyCandidate进行比较，如果没有相等的，则表示没有发生内存泄漏，直接调用注释5处的代码返回一个没有泄漏的分析结果AnalysisResult对象。如果找到了相等的，则表示发生了内存泄漏，执行注释6处的代码findLeakTrace()方法返回一个有泄漏分析结果的AnalysisResult对象。

条理清晰！

### 发送泄漏通知

回顾代码：

HeapAnalyzerService#onHandleIntentInForeground

```
  @Override protected void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    // 1，新建一个分析对象
    HeapAnalyzer heapAnalyzer =
        new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);

    // 2，检查泄漏，并返回结果
    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
        heapDump.computeRetainedHeapSize);
        
    // 3，发送结果
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
```

注释2处，拿到了 AnalysisResult 对象的分析结果，带有泄漏节点和泄漏路径等信息；
在注释3处，会把结果发送到通知栏：

引用 [Android主流三方库源码分析（六、深入理解Leakcanary源码）](https://juejin.cn/post/6844904070977749005#heading-20) 中的相关分析：

> AbstractAnalysisResultService.sendResultToListener()方法，很明显，这里AbstractAnalysisResultService的实现类就是我们刚开始分析的用于展示泄漏路径信息的DisplayLeakService对象。在里面直接创建一个由PendingIntent构建的泄漏通知用于供用户点击去展示详细的泄漏界面DisplayLeakActivity。核心代码如下所示：

> ```
> public class DisplayLeakService extends AbstractAnalysisResultService {

>     @Override
>     protected final void onHeapAnalyzed(@NonNull AnalyzedHeap analyzedHeap) {
>     
>         ...
>         
>         boolean resultSaved = false;
>         boolean shouldSaveResult = result.leakFound || result.failure != null;
>         if (shouldSaveResult) {
>             heapDump = renameHeapdump(heapDump);
>             // 1
>             resultSaved = saveResult(heapDump, result);
>         }
>         
>         if (!shouldSaveResult) {
>             ...
>             showNotification(null, contentTitle, contentText);
>         } else if (resultSaved) {
>             ...
>             // 2
>             PendingIntent pendingIntent =
>                 DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);

>             ...
>             
>             showNotification(pendingIntent, contentTitle, contentText);
>         } else {
>              onAnalysisResultFailure(getString(R.string.leak_canary_could_not_save_text));
>         }
>     
>     ...
> }

> @Override protected final void onAnalysisResultFailure(String failureMessage) {
>     super.onAnalysisResultFailure(failureMessage);
>     String failureTitle = getString(R.string.leak_canary_result_failure_title);
>     showNotification(null, failureTitle, failureMessage);
> }
> ```

> 可以看到，只要当分析的堆信息文件保存成功之后，即在注释1处返回的resultSaved为true时，才会执行注释2处的逻辑，即创建一个供用户点击跳转到DisplayLeakActivity的延时通知。

### DisplayLeakService

上述出现的 DisplayLeakService 类，该链接 [从 LeakCanary 探究线上内存泄漏检测方案](https://juejin.cn/post/6949759784253915172/) 有另一种解析：

分析结果 AnalysisResult 将交由一个 listener 组件处理，这个组件便是 DisplayLeakService ，在 DisplayLeakService 中有一段比较关键的代码:


```
  @Override protected final void onHeapAnalyzed(HeapDump heapDump, AnalysisResult result) {
    //对dump文件进行重命名
    heapDump = renameHeapdump(heapDump);
    //将AnalysisResult对象保存在xxx.hprof.result文件中
    resultSaved = saveResult(heapDump, result);
    ...
    PendingIntent = DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);
    // New notification id every second.
    int notificationId = (int) (SystemClock.uptimeMillis() / 1000);
    showNotification(this, contentTitle, contentText, pendingIntent, notificationId);
}

  private boolean saveResult(HeapDump heapDump, AnalysisResult result) {
    File resultFile = new File(heapDump.heapDumpFile.getParentFile(),
        heapDump.heapDumpFile.getName() + ".result");
    FileOutputStream fos = null;
    try {
      fos = new FileOutputStream(resultFile);
      ObjectOutputStream oos = new ObjectOutputStream(fos);
      oos.writeObject(heapDump);
      oos.writeObject(result);
      return true;
    } catch (IOException e) {
      CanaryLog.d(e, "Could not save leak analysis result to disk.");
    } finally {
      if (fos != null) {
        try {
          fos.close();
        } catch (IOException ignored) {
        }
      }
    }
    return false;
  }
```

在服务组件中，AnalysisResult对象被写进了 xxx.hprof.result 文件中。同时服务将显示一个 Notification，在 Notification 点击后将通过 DisplayLeakActivity 显示泄漏信息。

#### 泄漏引用链的显示

引用 [从 LeakCanary 探究线上内存泄漏检测方案](https://juejin.cn/post/6949759784253915172/) 的解析：

> ```
> //:DisplayLeakActivity.java

>   @Override protected void onResume() {
>     super.onResume();
>     LoadLeaks.load(this, getLeakDirectoryProvider(this));
>   }
> ```

> 再看看 LoadLeaks#load();

> ```
> //: LoadLeaks.java
> //LoadLeaks是runnable的子类
>     
>     static final List<LoadLeaks> inFlight = new ArrayList<>();
>     static final Executor backgroundExecutor = newSingleThreadExecutor("LoadLeaks");
>     
>     static void load(DisplayLeakActivity activity, LeakDirectoryProvider leakDirectoryProvider) {
>       LoadLeaks loadLeaks = new LoadLeaks(activity, leakDirectoryProvider);
>       inFlight.add(loadLeaks);
>       backgroundExecutor.execute(loadLeaks);
>     }
>     
>       @Override public void run() {
>       final List<Leak> leaks = new ArrayList<>();
>       List<File> files = leakDirectoryProvider.listFiles(new FilenameFilter() {
>         @Override public boolean accept(File dir, String filename) {
>           return filename.endsWith(".result");
>         }
>       });
>       for (File resultFile : files) {
>         FileInputStream fis = new FileInputStream(resultFile);
>         ObjectInputStream ois = new ObjectInputStream(fis);
>         HeapDump heapDump = (HeapDump) ois.readObject();
>         AnalysisResult result = (AnalysisResult) ois.readObject();
>         leaks.add(new Leak(heapDump, result, resultFile));
>      
>         mainHandler.post(new Runnable() {
>         @Override public void run() {
>           inFlight.remove(LoadLeaks.this);
>           if (activityOrNull != null) {
>             activityOrNull.leaks = leaks;
>             activityOrNull.updateUi();
>           }
>         }
>       });
> ```

> DisplayLeakActivity在 onResume 方法中，使用线程池读取所有的 xxx.prof.result 文件中的 AnalysisResult 对象，并使用 handler#post() 在主线程将它们加入到 Activity的成员变量 leaks 中，同时刷新 Activity 界面。

> 为了达到较好的显示效果，显示时会对引用链当中的节点信息进行格式上的美化，将字符串拼接成 html 格式显示, 具体逻辑可查看 DisplayLeakAdapter类。

### 一图流总结

引用 [Android主流三方库源码分析（六、深入理解Leakcanary源码）](https://juejin.cn/post/6844904070977749005#heading-20) 的图解：

![](/img/media/16214134386865/16214212255509.jpg)

### 线上使用的思路

引用 [从 LeakCanary 探究线上内存泄漏检测方案](https://juejin.cn/post/6949759784253915172/)：

> LeakCanary在判定有内存泄漏时，首先会生成一个内存快照文件(.hprof文件)，这个快照文件通常有10+M，然后根据 referenceKey 找出泄漏实例，再在快照堆中使用BFS找到实例所在的节点，并以此节点信息反向生成最小引用链。在生成引用链后，将其保存在 AnalysisResult 对象当中，然后将AnalysisResult对象写入.hporf.result文件，此时的 「**.hprof.result文件只有几十KB大小**」。最后，在 DisplayLeakActivity 的 onResume 中读取所有的 .hprof.result文件并显示在界面上。

> LeakCanary 直接在线上使用会有什么样的问题，如何改进呢？
> 理解了 LeakCanary 判定对象泄漏后所做的工作后就不难知道，直接将 LeakCanary 应用于线上会有如下一些问题：

> 1，每次内存泄漏以后，都会生成一个.hprof文件，然后解析，并将结果写入.hprof.result。增加手机负担，引起手机卡顿等问题。
> 2，同样的泄漏问题，会重复生成 .hprof 文件，重复分析并写入磁盘。
> 3，.hprof文件较大，信息回捞成问题。

> 那应该如何解决上述问题呢？

> 1，可以根据手机信息来设定一个内存阈值 M ，当已使用内存小于 M 时，如果此时有内存泄漏，只将泄漏对象的信息放入内存当中保存，不生成.hprof文件。当已使用大于 M 时，生成.hprof文件；当然，也可以判断泄漏对象数量大于某个规定的数值时，生成.hprof文件并分析，此时的分析结果应当包含一个或多个泄漏对象引用链的信息。
> 2，当引用链路相同时，根据实际情况看去重。
> 3，不直接回捞.hprof文件，可以选择回捞.hprof.result文件，或者在本地对.hprof.result进行进一步的整合、去重与优化后再回捞。
> 4，引用链上的信息是混淆后的，所以在需要做版本控制并维护混淆mapping文件做进一步解析。

简单的说：
1，内存阈值，以此选择保存在内存或文件；泄漏数量阈值，超过才生成文件；
2，去重；
3，小体积文件；
4，混淆处理；

### 另一种线上使用的思路

高频采集，去重？好像是卡顿的线上方案。。。

### 2.0 版本的新特性

hencoder 视频课的最后10分钟。

#### Kotlin 实现

#### 免启动

找到该路径下的文件：

com.squareup.leakcanary/
leakcanary-object-watcher-android/2.2/
.../leakcanary-object-watcher-android-2.2-sources.jar!/
leakcanary/internal/**AppWatcherInstaller.kt**

```
internal sealed class AppWatcherInstaller : ContentProvider() {

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    InternalAppWatcher.install(application)
    return true
  }
```

利用了 ContentProvider 会自动加载的机制。

