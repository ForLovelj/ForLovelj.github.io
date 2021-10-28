---
layout: post
title: LeakCanary源码分析
author: clow
date: 2020-10-05 21:02:05
categories:
- Android
tags: Android
---
## 1. 序
基于`com.squareup.leakcanary:leakcanary-android:2.5`
> LeakCanary 是一个适用于 Android 的内存泄漏检测库，其具有一种独特的能力，可以缩小每个泄漏的原因，从而帮助开发人员显着减少OutOfMemoryError崩溃。

LeakCanary 自动检测以下对象的泄漏：

- 销毁的Activity实例
- 销毁的Fragment实例
- 销毁的片段View实例
- 清除ViewModel实例

那么LeakCanary是通过什么方式来检查内存泄漏的呢？

## 2. LeakCanary检测内存泄漏的原理
在分析LeakCanary检测内存泄漏的原理之前我们先来看下Java中存在的4种引用类型：
### 2.1 Java引用分类
- 强引用：平时常用的引用类型，JVM发生OOM也不会回收这部分引用。
- 软引用(SoftReference)：对于软引用关联着的对象，在JVM应用即将发生内存溢出异常之前，将会把这些软引用关联的对象列进去回收对象范围之中进行第二次回收。如果这次回收之后还是没有足够的内存，才会抛出内存溢出异常。
- 弱引用(WeakReference)：被弱引用关联的对象只能生存到下一次垃圾收集发生之前，简言之就是：一旦发生GC必定回收被弱引用关联的对象，不管当前的内存是否足够。也就是弱引用只能活到下次GC之时。
- 虚引用(PhantomReference)：一个对象是否关联到虚引用，完全不会影响该对象的生命周期，也无法通过虚引用来获取一个对象的实例(PhantomReference覆盖了Reference#get()并且总是返回null)。为对象设置一个虚引用的唯一目的是：能在此对象被垃圾收集器回收的时候收到一个系统通知。

### 2.2 Refercence及ReferenceQueue:
```
public abstract class Reference<T> {
    // 代表这个引用关联的对象，被回收时置null
    private T referent;
    //引用队列，保存即将被回收的reference对象
    volatile ReferenceQueue<? super T> queue;
    //引用链表中的下一个元素
    volatile Reference next;
    //基于状态表示不同链表中的下一个待处理的对象
    private transient Reference<T> discovered;

    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
}  

```
- `referent`代表这个引用关联的对象
- `queue`引用队列。如果引用关联的对象即将被垃圾回收器回收，那个该对象会被添加到这个队列中。在关联对象时可以不指定队列，那么`queue`的值就是- `ReferenceQueue.NULL`，后续在入队过程中，检测到当前引用拥有的时这个队列，会直接返回false。
- `next`引用链表中的下一个元素。虽然引用有引用队列，但是引用是通过这个来形成单向链表的，并不依赖`ReferenceQueue`，`ReferenceQueue`中只保存了这个链表的`head`节点。
- `discovered`基于状态表示不同链表中的下一个待处理的对象，主要是pending-reference列表的下一个元素，通过JVM直接调用赋值。

对于这些引用的处理，在java后台会有一个专门的守护线程`ReferenceHandler`来执行，我们这儿不做深究。

### 2.3 LeakCanary检测内存泄漏的原理
通过上面对java引用类型和`Refercence`及`ReferenceQueue`的了解，我们可以知道：如果我们用一个
弱引用`WeakReference`来保存对象，并且在实例化`WeakReference`时传入一个`ReferenceQueue`，那么我们在调用系统`gc`方法时这个对象就会被放入`ReferenceQueue`中等待系统回收，我们此时遍历`ReferenceQueue`：

- 如果内部包含我们之前保存的对象，那么是不是可以认为这个对象已经被系统回收而没有出现内存泄漏。
- 如果内部未包含我们之前保存的对象，那么是不是可以认为已经发生了内存泄漏。

没错，这就是`LeakCanary`检测内存泄漏的核心原理，`LeakCanary`内部用到了`Refercence`及`ReferenceQueue`来实现对对象是否被回收的监听。

接下来我们通过分析源码来看一看`LeakCanary`是如何检测内存泄漏，并发出内存泄漏通知的。
## 3. LeakCanary源码解析
为了更好的对LeakCanary源码进行分部解析，我们先对LeakCanary实现内存泄漏的整体过程做一个概括：

1. 初始化。
2. 添加相关监听对象销毁监听，LeakCanary会默认监听Activity、Fragment、Fragment的View、ViewModel是否回收。
3. 收到销毁回调后，根据要回收对象创建KeyedWeakReference和ReferenceQueue，并关联。
4. 延迟5秒检查相关对象是否被回收。
5. 如果没有被回收就通过dump heap获取hprof文件。
6. 通过Shark库解析hprof文件，获取泄漏对象，被计算泄漏对象到GC roots的最短路径。
7. 合并多个泄漏路径并输出分析结果。
8. 将结果展示到可视化界面。
### 3.1 LeakCanary初始化
LeakCanary使用了ContentProvider来自动初始化。我们都知道ContentProvider的onCreate的调用时机介于Application的attachBaseContext和onCreate之间，Provider的onCreate优先于Application的onCreate执行。此时的Application已经创建成功，而Provider里的context正是Application的对象。目前很多三方库都采用这种无感知的初始化方式了。
```
//manifest
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.squareup.leakcanary.objectwatcher" >

    <uses-sdk android:minSdkVersion="14" />

    <application>
        <provider
            android:name="leakcanary.internal.AppWatcherInstaller$MainProcess"
            android:authorities="${applicationId}.leakcanary-installer"
            android:enabled="@bool/leak_canary_watcher_auto_install"
            android:exported="false" />
    </application>

</manifest>
```
```
internal sealed class AppWatcherInstaller : ContentProvider() {

    ...
    override fun onCreate(): Boolean {
        val application = context!!.applicationContext as Application
        //最终内部调用的是InternalAppWatcher.install(application)完成初始化
        AppWatcher.manualInstall(application)
        return true
    }
    ...
}
```

我们可以看到`AppWatcherInstaller`初始话内部调用了`AppWatcher.manualInstall(application)`，最终内部调用的是InternalAppWatcher.install(application)完成初始化:
```
    //AppWatcherInstaller
    ...
    private val checkRetainedExecutor = Executor {
        //主线程延时AppWatcher.config.watchDurationMilli执行任务，默认为5s
        mainHandler.postDelayed(it, AppWatcher.config.watchDurationMillis)
    }

    //实例化ObjectWatcher用于观测对象是否回收
    val objectWatcher = ObjectWatcher(
        clock = clock,
        checkRetainedExecutor = checkRetainedExecutor,
        isEnabled = { true }
    )

    ...
  //InternalAppWatcher
  fun install(application: Application) {
      //校验是否是主线程
    checkMainThread()
    //如果已经完成过初始化，直接return
    if (this::application.isInitialized) {
      return
    }
    InternalAppWatcher.application = application
    if (isDebuggableBuild) {
      SharkLog.logger = DefaultCanaryLog()
    }
    //读取配置信息
    val configProvider = { AppWatcher.config }
    //注册Activity destroy监听
    ActivityDestroyWatcher.install(application, objectWatcher, configProvider)
    //注册Fragment destroy监听
    FragmentDestroyWatcher.install(application, objectWatcher, configProvider)
    //InternalleakCanary初始化（反射）
    onAppWatcherInstalled(application)
  }

  ...
```
### 3.2 观察是否发生内存泄漏
我们以`activity`为例来探究`LeakCanary`是如何进行内存泄漏检测的实现的。我们接着来看`ActivityDestroyWatcher.install()`方法：
```
//ActivityDestroyWatcher
...
private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        if (configProvider().watchActivities) {
          //通过objectWatcher监听改activity是否被销毁回收
          objectWatcher.watch(
              activity, "${activity::class.java.name} received Activity#onDestroy() callback"
          )
        }
      }
    }

  companion object {
    fun install(
      application: Application,
      objectWatcher: ObjectWatcher,
      configProvider: () -> Config
    ) {
      val activityDestroyWatcher =
        ActivityDestroyWatcher(objectWatcher, configProvider)
      application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
    }
  }
  ...
```
可以看到`ActivityDestroyWatcher.install()`内部实际上是注册了一个`ActivityLifecycle`监听，在`onActivityDestroyed`回调中调用了`objectWatcher.watch()`方法来观察当前`activity`是否发生了内存泄漏。

接着我们来看下`objectWatcher.watch()`内部都干了什么：
```
 /**
   * Watches the provided [watchedObject].
   *
   * @param description Describes why the object is watched.
   */
  @Synchronized fun watch(
    watchedObject: Any,
    description: String
  ) {
    if (!isEnabled()) {
      return
    }
    //1.清空queue，即移除之前已回收的引用
    removeWeaklyReachableObjects()
    //生成随机的key值，用来作为保存由检测对象创建的KeyedWeakReference的key
    val key = UUID.randomUUID()
        .toString()
    //记录当前时间
    val watchUptimeMillis = clock.uptimeMillis()
    //将当前Activity对象封装成KeyedWeakReference，并关联引用队列queue
    //KeyedWeakReference继承自WeakReference，封装了用于监听对象的辅助信息
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
      "Watching " +
          (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
          (if (description.isNotEmpty()) " ($description)" else "") +
          " with key $key"
    }

    //将弱引用reference存入监听列表watchedObjects
    watchedObjects[key] = reference
    //2.进行一次后台检查任务，判断引用对象是否未被回收
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
  }
```
调用`removeWeaklyReachableObjects()`清空queue，即移除之前已回收的引用。

我们来接着看：
```
  private fun removeWeaklyReachableObjects() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
    do {
      //遍历引用队列
      //如果activity被回收了，包含该activity的KeyedWeakReference就会在该queue中
      ref = queue.poll() as KeyedWeakReference?
      if (ref != null) {
        //根据key把map中对象移除
        watchedObjects.remove(ref.key)
      }
    } while (ref != null)
  }
```
这个方法的作用就是判断哪些对象已经被加入`ReferenceQueuee`等待回收了，然后从本地`watchedObjects`中将这些对象移除掉。

第一次调用是清除之前的已回收对象，后面在`moveToRetained`中还会再次调用该方法判断引用是否正常回收。

我们接着来看下`checkRetainedExecutor.execute { moveToRetained(key) }`方法，执行一个延时任务，5秒后判断引用对象是否未被回收。为什么是5秒，我们前面在介绍`AppWatcherInstaller`类的时候已经说过。
```
  @Synchronized private fun moveToRetained(key: String) {
    //遍历引用队列，并将引用队列中的引用从监听列表watchedObjects中移除
    removeWeaklyReachableObjects()
    //若对象未能成功移除，则表明引用对象可能存在内存泄漏
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
      //可能发生了泄漏。通知注册了的Listener
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
  }
```
在activity destroy 5s后，我们走到了`moveToRetained`方法，先是调用`removeWeaklyReachableObjects()`方法将等待回收的对象从`watchedObjects`中移除，然后通过指定的`key`查找对应的`KeyedWeakReference`（这个KeyedWeakReference对应的就是我们activity销毁时，Activity对象封装成KeyedWeakReference）是否在`watchedObjects`还存在。
- 存在：可能发生了泄漏，通知注册了的Listener.
- 不存在：nothing

### 3.3 发起GC再次检测
上面说到发现内存泄漏后会通知注册了的Listener，这个Listener实际上就是`InternalLeakCanary`:
```
//InternalLeakCanary
override fun onObjectRetained() = scheduleRetainedObjectCheck()

  fun scheduleRetainedObjectCheck() {
    if (this::heapDumpTrigger.isInitialized) {
      heapDumpTrigger.scheduleRetainedObjectCheck()
    }
  }
```
可以看到，内部实际上是调用了`heapDumpTrigger.scheduleRetainedObjectCheck()`方法：
```
//HeapDumpTrigger

fun scheduleRetainedObjectCheck(
    delayMillis: Long = 0L
  ) {
    ...
    backgroundHandler.postDelayed({
      checkScheduledAt = 0
      checkRetainedObjects()
    }, delayMillis)
  }

  private fun checkRetainedObjects() {
    ...
    //还保留的对象数
    var retainedReferenceCount = objectWatcher.retainedObjectCount
    //执行一次GC，再更新剩余保留的对象数
    if (retainedReferenceCount > 0) {
      gcTrigger.runGc()
      retainedReferenceCount = objectWatcher.retainedObjectCount
    }
    //小于5个不去dump
    if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return
    ...
    //dumpHeap来创建hprof文件
    dumpHeap(retainedReferenceCount, retry = true)
  }

```
`dumpHeap()` 获取内存快照，生成hprof文件，来看下dumpHeap方法实现：
```
private fun dumpHeap(
    retainedReferenceCount: Int,
    retry: Boolean
  ) {
    saveResourceIdNamesToMemory()
    val heapDumpUptimeMillis = SystemClock.uptimeMillis()
    KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
    ////获取当前内存快照hprof文件
    when (val heapDumpResult = heapDumper.dumpHeap()) {
      is NoHeapDump -> {
          //失败处理
        if (retry) {
          SharkLog.d { "Failed to dump heap, will retry in $WAIT_AFTER_DUMP_FAILED_MILLIS ms" }
          scheduleRetainedObjectCheck(
              delayMillis = WAIT_AFTER_DUMP_FAILED_MILLIS
          )
        } else {
          SharkLog.d { "Failed to dump heap, will not automatically retry" }
        }
        showRetainedCountNotification(
            objectCount = retainedReferenceCount,
            contentText = application.getString(
                R.string.leak_canary_notification_retained_dump_failed
            )
        )
      }
      is HeapDump -> {
        lastDisplayedRetainedObjectCount = 0
        lastHeapDumpUptimeMillis = SystemClock.uptimeMillis()
        //清理之前注册的监听
        objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
        ////开启hprof分析Service，解析hprof文件生成报告
        HeapAnalyzerService.runAnalysis(
            context = application,
            heapDumpFile = heapDumpResult.file,
            heapDumpDurationMillis = heapDumpResult.durationMillis
        )
      }
    }
  }
```
### 3.4 hprof文件解析
在上面讲到的内存泄漏回调处理中，生成了hprof文件，并开启一个服务来解析该文件。在Service的onHandleIntentInForeground回调方法中进行hprof文件解析:
```
//HeapAnalyzerService
override fun onHandleIntentInForeground(intent: Intent?) {
    if (intent == null || !intent.hasExtra(HEAPDUMP_FILE_EXTRA)) {
      SharkLog.d { "HeapAnalyzerService received a null or empty intent, ignoring." }
      return
    }

    // Since we're running in the main process we should be careful not to impact it.
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)
    val heapDumpFile = intent.getSerializableExtra(HEAPDUMP_FILE_EXTRA) as File

    val config = LeakCanary.config
    //解析hporf文件，返回heapAnalysis
    val heapAnalysis = if (heapDumpFile.exists()) {
      analyzeHeap(heapDumpFile, config)
    } else {
      missingFileFailure(heapDumpFile)
    }
    onAnalysisProgress(REPORTING_HEAP_ANALYSIS)
    //解析完成回调，这个回调函数的实现类是DefaultOnHeapAnalyzedListener
    config.onHeapAnalyzedListener.onHeapAnalyzed(heapAnalysis)
  }
```
调用链：HeapAnalyzerService.analyzeHeap–>HeapAnalyzer.analyze。该方法实现了解析hprof文件找到内存泄漏对象，并计算对象到GC roots的最短路径，输出报告。
```
 /**
   * Searches the heap dump for leaking instances and then computes the shortest strong reference
   * path from those instances to the GC roots.
   */
  fun analyze(
    heapDumpFile: File,
    leakingObjectFinder: LeakingObjectFinder,
    referenceMatchers: List<ReferenceMatcher> = emptyList(),
    computeRetainedHeapSize: Boolean = false,
    objectInspectors: List<ObjectInspector> = emptyList(),
    metadataExtractor: MetadataExtractor = MetadataExtractor.NO_OP,
    proguardMapping: ProguardMapping? = null
  ): HeapAnalysis {

    ...

    return try {
      //回调PARSING_HEAP_DUMP解析状态
      listener.onAnalysisProgress(PARSING_HEAP_DUMP)
      val sourceProvider = ConstantMemoryMetricsDualSourceProvider(FileSourceProvider(heapDumpFile))
      //开始解析hprof文件
      sourceProvider.openHeapGraph(proguardMapping).use { graph ->
        //从文件中解析获取对象关系图结构graph
        //并获取图中的所有GC roots根节点

        //创建FindLeakInput对象
        val helpers =
          FindLeakInput(graph, referenceMatchers, computeRetainedHeapSize, objectInspectors)
        //查找内存泄漏对象
        val result = helpers.analyzeGraph(
            metadataExtractor, leakingObjectFinder, heapDumpFile, analysisStartNanoTime
        )
        val lruCacheStats = (graph as HprofHeapGraph).lruCacheStats()
        val randomAccessStats =
          "RandomAccess[" +
              "bytes=${sourceProvider.randomAccessByteReads}," +
              "reads=${sourceProvider.randomAccessReadCount}," +
              "travel=${sourceProvider.randomAccessByteTravel}," +
              "range=${sourceProvider.byteTravelRange}," +
              "size=${heapDumpFile.length()}" +
              "]"
        val stats = "$lruCacheStats $randomAccessStats"
        result.copy(metadata = result.metadata + ("Stats" to stats))
      }
    } catch (exception: Throwable) {
      HeapAnalysisFailure(
          heapDumpFile = heapDumpFile,
          createdAtTimeMillis = System.currentTimeMillis(),
          analysisDurationMillis = since(analysisStartNanoTime),
          exception = HeapAnalysisException(exception)
      )
    }
  }
```
接着我们来看下`FindLeakInput.analyzeGraph`方法查找内存泄漏对象：
```
private fun FindLeakInput.analyzeGraph(
    metadataExtractor: MetadataExtractor,
    leakingObjectFinder: LeakingObjectFinder,
    heapDumpFile: File,
    analysisStartNanoTime: Long
  ): HeapAnalysisSuccess {
    ...
    //通过过滤graph中的KeyedWeakReference类型对象来
    //找到对应的内存泄漏对象
    val leakingObjectIds = leakingObjectFinder.findLeakingObjectIds(graph)
    //计算内存泄漏对象到GC roots的路径
    val (applicationLeaks, libraryLeaks) = findLeaks(leakingObjectIds)
    //输出最终hprof分析结果
    return HeapAnalysisSuccess(
        heapDumpFile = heapDumpFile,
        createdAtTimeMillis = System.currentTimeMillis(),
        analysisDurationMillis = since(analysisStartNanoTime),
        metadata = metadata,
        applicationLeaks = applicationLeaks,
        libraryLeaks = libraryLeaks
    )
  }
```
最终在可视化界面中将hprof分析结果`HeapAnalysisSuccess`展示出来。

### 3.5 将内存泄漏信息通知用户
`onHeapAnalyzedListener.onHeapAnalyzed`的实现类是`DefaultOnHeapAnalyzedListener`，来看下具体实现：
```
override fun onHeapAnalyzed(heapAnalysis: HeapAnalysis) {
    SharkLog.d { "\u200B\n${LeakTraceWrapper.wrap(heapAnalysis.toString(), 120)}" }

    val id = LeaksDbHelper(application).writableDatabase.use { db ->
      HeapAnalysisTable.insert(db, heapAnalysis)
    }

    val (contentTitle, screenToShow) = when (heapAnalysis) {
      is HeapAnalysisFailure -> application.getString(
          R.string.leak_canary_analysis_failed
      ) to HeapAnalysisFailureScreen(id)
      is HeapAnalysisSuccess -> {
        val retainedObjectCount = heapAnalysis.allLeaks.sumBy { it.leakTraces.size }
        val leakTypeCount = heapAnalysis.applicationLeaks.size + heapAnalysis.libraryLeaks.size
        application.getString(
            R.string.leak_canary_analysis_success_notification, retainedObjectCount, leakTypeCount
        ) to HeapDumpScreen(id)
      }
    }

    if (InternalLeakCanary.formFactor == TV) {
        //toast形式
      showToast(heapAnalysis)
      printIntentInfo()
    } else {
        //通知形式
      showNotification(screenToShow, contentTitle)
    }
  }
```

## 4. 总结
我们以`Activity为`例，来总结下`LeakCanary`检测内存泄漏过程：
1. 监听`Activity`的`onDestroy`事件
2. 在`onDestroy`事件回调中使用`objectWatcher.watch()`创建引用activity的`KeyedWeakReference`对象，并关联ReferenceQueue。
3. 延时5秒调用`moveToRetained()`检查目标对象是否回收。
4. 未回收则手动触发`gc`再次检测未回收对象数量。
5. 未回收对象大于5个则开启服务，dump heap获取内存快照hprof文件。
6. 解析hprof文件根据`KeyedWeakReference`类型过滤找到内存泄漏对象。
7. 计算对象到GC roots的最短路径，并合并所有最短路径为树结构。
8. 输出分析结果，并根据分析结果展示到可视化页面。

![LeanCanary流程图](https://ForLovelj.github.io/img/LeanCanary流程图.png)