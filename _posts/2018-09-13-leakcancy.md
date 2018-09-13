---
layout:     post
title:      LeakCanary内存优化学习
subtitle:   内存检测工具学习
date:       2018-09-13
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Java
    - Android
---

## Android常见的内存泄漏

### 单例造成的内存泄漏

单例错误写法

{% highlight java %}
public class SingletonActivityContext {
    private static SingletonActivityContext instance;
	private Context context;
	
	private SingletonActivityContext(Context context) {
        this.context = context;    
    }

    public static SingletonActivityContext getInstance(Context context) {
        if(instance == null) {
            instance = new SingletonActivityContext(context);
        }
        return instance;
    }
}
{% endhighlight %}

使用内部静态类的单例会使这个类的生命周期与整个APP的生命周期一样长，如果使用不当，很容易造成内存泄漏。最简单的处理方法就是传入Application的context，就不会内存泄漏了。

###  非静态内部类创建静态实例造成的内存泄漏

{% highlight java %}
public class StaticLeakActivity extends Activity {
    private static noneStaticClass mResource = null;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if(mResource == null) {
            mResource = new nonoStaticClass();
        }
    }

    private class noneStaticClass {
    }
}
{% endhighlight %}

在一个频繁启动的Activity中使用一个静态变量持有一个非静态内部类的好处是，可以减少创建非静态内部类，但造成的问题是产生了内存泄漏。因为非静态内部类持有Activity的引用，而```mResource```的生命周期与整个App的声明周期一样长，并且持有非静态内部类的引用，造成Activity不能被回收，产生内存泄漏。

正确的做法是将这个非静态内部类改为静态内部类。

### handler造成内存泄漏

{% highlight java %}
public class HandlerLeakActivity extends Activity {
    private final Handler mLeakyHandler = new Handler() {
        @Override
        public void handlerMessage(Message msg) {
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mLeakyHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
            }
        }, 1000 * 60 * 10); // 延迟10分钟发送消息
    }
}
{% endhighlight %}

在```HandlerLeakActivity```中我们的handler延迟10分钟发送消息，在10分钟之前，这个消息会放到messageQuene当中。如果我们在10分钟之前调用了```HandlerLeakActivity```的```finish()```方法，message持有handler的引用，handler又持有HandlerLeakActivity的引用，那么HandlerLeakActivity就不会被回收，产生了内存泄漏。

正确做法：1）将handler声明为静态的；2）通过弱引用的方式持有handler。

### 线程所造成的内存泄漏

正确写法，将异步执行的类都写成static，防止内部类持有外部类的匿名引用，避免内存泄漏。

{% highlight java %}
public class ThreadLeakActivity extends Activity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        testThreadLeak();
    }

    private void testThreadLeak() {
        (AsyncTask) (params) -> {
            SystemClock.sleep(10 * 1000);
            return null;
        }.execute();

        new Thread((Runnable) () -> {
            SystemClock.sleep(10 * 1000);
        }).start();
    }

    static class MyAsyncTask extends AsyncTask<Void, void, Void> {
        private WeakReference<Context> weakReference;
        public MyAsyncTask(Context context) {
            weakReference = new WeakReference;
        }

        @Override
        protected Void doInbackground(Void... params) {
            SystemClock.sleep(10 * 1000);
            return null;
        }

        @Override
        protected void onPostExecute(Void aVoid) {
            super.onPostExecute(aVoid);
            TheadLeakyActivity activity = (ThreadLeakyActivity) weakReference;
            if(activity != null) {
                //...
            }
        }
    }

    static class MyRunnable implements Runnable {
        @Override
        public void run() {
            SystemClock.sleep(10 * 1000);
        }
    }
}
{% endhighlight %}

### WebView造成的内存泄漏

{% highlight java %}
public class WebviewLeakActivity extends AppCompatActivity {
    private Webview mWebview;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mWebview = (WebView) findViewById(R.id.wv_show);
        mWebview.loadUrl("http://www.baidu.com");
    }

    @Override
    protected void onDestroy() {
        destroyWebView();
        android.os.Process.killProcess(android.os.Process.myPid);
        super.onDestroy();
    }

    private void destroyWebView() {
        if(mWebview != null) {
            mWebview.pauseTimers();
            mWebview.removeAllViews();
            mWebview.destroy();
        }
    }
}
{% endhighlight %}

WebView加载页面时会使用堆内存存储页面元素。正确的避免WebView的做法是，将WebView放到一个单独的进程中，在监测内存占用过大时调用```killProcess()```方法杀掉这个进程。

## LeakCanary原理

1. Activity Destroy之后将它放在一个WeakReference中。
2. 将这个WeakReference关联到一个ReferenceQueue中。
3. 查看ReferenceQueue是否存在Activity的引用。
4. 如果Activity泄露了，Dump出heap信息，然后再去分析泄露路径。

> 四种引用回顾：
> 
> 1）强引用（StrongReference）：我们平常创建的对象引用，在内存不足时，jvm宁愿抛出oom，也不会轻易回收强引用。
> 
> 2）软引用（SoftReference）:当内存空间足够时，软引用和强引用是一样的，只有在内存空间不足时，会回收软引用。可以和引用队列关联起来。
> 
> 3）弱引用（WeakReference）:当gc扫描内存空间时，不会区分内存够不够，遇到弱引用直接回收。可以和引用队列关联起来。
> 
> 4）虚引用:它不会决定一个对象的生命周期。换句话说就是虚引用持有一个对象时，gc可以在任何时候回收它。

### ReferenceQueue

* 软引用/弱引用
* 对象被垃圾回收，Java虚拟机就会把这个引用加入到与之关联的引用队列中。

### LeakCanary源码分析

按照文档说明，LeakCanary只需在Application的onCreate()方法中写一句```LeakCanary.install(this);```即可。那我们先看```install()```方法。

{% highlight java %}
public final class LeakCanary {

    /**
     * Creates a {@link RefWatcher} that works out of the box, and starts watching activity
     * references (on ICS+).
     */
    public static RefWatcher install(Application application) {
        return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
            .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
            .buildAndInstall();
    }
    ...
}
{% endhighlight %}

这个install方法返回一个RefWatcher类，用来启动Activity的RefWatcher类的。在Activity调用完onCreate(savedInstanceState)方法之后，RefWatcher会探测Activity的内存泄漏。这个install方法最终会返回一个buildAndInstall()方法，我们进去看一下它的实现。

{% highlight java %}
// AndroidRefWatcherBuilder.java
public RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
        throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
        if (watchActivities) {
            ActivityRefWatcher.install(context, refWatcher);
        }
        if (watchFragments) {
            FragmentRefWatcher.Helper.install(context, refWatcher);
        }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
}
{% endhighlight %}

这个方法首先创建了一个RefWatcher对象，然后通过判断这个refWatcher对象是否可用并且按需对Activity和Fragment进行监测。这个方法的返回值是一个refWatcher，用来检测内存泄漏的。

{% highlight java %}
//ActivityRefWatcher.install(context, refWatcher);
public static void install(Context context, RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);

    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
}

//FragmentRefWatcher.Helper.install(context, refWatcher);
public static void install(Context context, RefWatcher refWatcher) {
    List<FragmentRefWatcher> fragmentRefWatchers = new ArrayList<>();

    if (SDK_INT >= O) {
      fragmentRefWatchers.add(new AndroidOFragmentRefWatcher(refWatcher));
    }

    try {
        Class<?> fragmentRefWatcherClass = Class.forName(SUPPORT_FRAGMENT_REF_WATCHER_CLASS_NAME);
        Constructor<?> constructor =
            fragmentRefWatcherClass.getDeclaredConstructor(RefWatcher.class);
        FragmentRefWatcher supportFragmentRefWatcher =
            (FragmentRefWatcher) constructor.newInstance(refWatcher);
        fragmentRefWatchers.add(supportFragmentRefWatcher);
    } catch (Exception ignored) {
    }

    if (fragmentRefWatchers.size() == 0) {
        return;
    }

    Helper helper = new Helper(fragmentRefWatchers);

    Application application = (Application) context.getApplicationContext();
    application.registerActivityLifecycleCallbacks(helper.activityLifecycleCallbacks);
}
{% endhighlight %}

先说一下```ActivityRefWatcher.install()```逻辑，首先获取Application的context，然后new一个ActivityRefWatcher，最后将ActivityRefWatcher的生命周期回调接口注册到application的Activity生命周期回调接口列表中。ActivityRefWatcher的lifecycleCallbacks接口并不神秘，它只是一些列在Activity各个声明周期会执行的方法。

{% highlight java %}
public interface ActivityLifecycleCallbacks {
    void onActivityCreated(Activity activity, Bundle savedInstanceState);
    void onActivityStarted(Activity activity);
    void onActivityResumed(Activity activity);
    void onActivityPaused(Activity activity);
    void onActivityStopped(Activity activity);
    void onActivitySaveInstanceState(Activity activity, Bundle outState);
    void onActivityDestroyed(Activity activity);
}
{% endhighlight %}

然后我们说一下```FragmentRefWatcher.Helper.install()```方法，首先在该方法中有一个列表存储了fragmentRefWatcher，当```SDK_INT >= O```（API level >= 26）时会单独加入一个AndroidOFragmentRefWatcher，然后利用反射创建了一个SupportFragmentRefWatcher对象并加入到fragmentRefWatcher列表中，如果上述过程发生错误程序直接返回，从而不对fragment的内存泄露进行监听，当FragmentRefWatcher成功加入列表后，最终将这个列表检测用到的生命周期回调接口注册到Application之中。

在创建ActivityRefWatcher时，有一个RefWatcher类非常重要，这个类实现了对Activity和Fragment生命周期内内存泄漏的监测，接下来我们着重看一下它的实现。

{% highlight java %}
public final class RefWatcher {
    public static final RefWatcher DISABLED = (new RefWatcherBuilder()).build();
    private final WatchExecutor watchExecutor; // 用于内存泄漏检测的类
    private final DebuggerControl debuggerControl; // 查询是否正在进行调试中，如果在调试中就不会进行内存泄漏检测的判断
    private final GcTrigger gcTrigger; // 在内存泄漏之前会给泄漏变量一次机会去调用GcTrigger方法中的gc，判断是否会被gc,若不gc就会给用户显示出来
    private final HeapDumper heapDumper; // 内存泄漏的堆文件
    private final Listener heapdumpListener; // 用于分析产生heap文件的一些回调
    private final Builder heapDumpBuilder; 
    private final Set<String> retainedKeys; // 持有待检测和已经内存泄漏对象引用的key
    private final ReferenceQueue<Object> queue;// 引用队列，用来判断弱引用持有的对象是否已经执行垃圾回收
}
{% endhighlight %}

分析完RefWatcher的成员变量后，我们分析一下RefWatcher的成员方法watch。

{% highlight java %}
public void watch(Object watchedReference, String referenceName) {
    if(this != DISABLED) {
        Preconditions.checkNotNull(watchedReference, "watchedReference");
        Preconditions.checkNotNull(referenceName, "referenceName");
        long watchStartNanoTime = System.nanoTime();
        String key = UUID.randomUUID().toString();
        this.retainedKeys.add(key);
        KeyedWeakReference reference = new KeyedWeakReference(watchedReference, key, referenceName, this.queue);
        this.ensureGoneAsync(watchStartNanoTime, reference);
    }
}
{% endhighlight %}

核心语句是这句```String key = UUID.randomUUID().toString();```，用处是对我们的reference加一个唯一的key值，这个key值在后面会有用。然后将这个key添加到成员变量retainedKeys列表中。接着根据观测对象和它对应的key值new了一个KeyedWeakReference弱引用对象。最后开启一个异步线程去分析这个对象引用。

接下来我们看一下异步线程```ensureGoneAsync()```方法中做了什么操作。

{% highlight java %}
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    this.watchExecutor.execute(new Retryable() {
        public Result run() {
            return RefWatcher.this.ensureGone(reference, watchStartNanoTime);
        }
    });
}
{% endhighlight %}

我们在到```ensureGone()```方法中去看一下。

{% highlight java %}
Result ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    // 用于计算当我们调用watch方法到gc垃圾回收一共花费的时间
    long watchDurationMs = TimeUnit.NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
    // 清除已经到达引用队列的弱引用，将已经回收的key从我们的引用队列里移除，剩下的key就是未被回收的对象
    this.removeWeaklyReachableReferences(); 
    if(this.debuggerControl.isDebuggerAttached()) {
        return Result.RETRY; // 如果已经进入调试状态，则不会进行内存泄漏的分析
    } else if(this.gone(reference)) {
        return Result.DONE; // 如果所有引用都已经被回收了，则返回DONE，此时未发生内存泄漏
    } else {
        this.gcTrigger.runGc(); // 如果当前还有对象未被回收，则再次调用gc，来进行手动的垃圾回收
        this.removeWeaklyReachableReferences(); // 清除到达引用队列的弱引用，与上次手动gc相对应
        if(!this.gone(reference)) { // 此时若能进到if语句中，说明了真发生了内存泄漏
            long startDumpHeap = System.nanoTime();
            long gcDurationMs = TimeUnit.NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
            File heapDumpFile = this.heapDumper.dumpHeap(); // 将内存泄漏信息保存到文件中
            if(heapDumpFile == HeapDumper.RETRY_LATER) {
                return Result.RETRY;
            }

            long heapDumpDurationMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
            HeapDump heapDump = this.heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key).referenceName(reference.name).watchDurationMs(watchDurationMs).gcDurationMs(gcDurationMs).heapDumpDurationMs(heapDumpDurationMs).build();
            this.heapdumpListener.analyze(heapDump); // 在这里会去分析内存泄漏的路径
        }

        return Result.DONE;
    }
}
{% endhighlight %}

这个函数从名字上看就是确保我们的Activity进入到GONE状态，换句话说就是Activity是否已经被回收了。我们继续看分析内存泄漏的方法```analyze()```方法。

{% highlight java %}
// HeapDump.java
public interface Listener {
    HeapDump.Listener NONE = new HeapDump.Listener() {
        public void analyze(HeapDump heapDump) {
        }
    };

    void analyze(HeapDump var1);
}
{% endhighlight %}

可以看出analyze方法是HeapDump的Listener接口的一个方法，我们继续看它的实现。

{% highlight java %}
// ServiceHeapDumpListener.java
@Override public void analyze(HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
}
{% endhighlight %}

继续进入runAnalysis()方法。

{% highlight java %}
// HeapAnalyzerService.java 继承与ForegroundService
public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    setEnabledBlocking(context, HeapAnalyzerService.class, true);
    setEnabledBlocking(context, listenerServiceClass, true);
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    ContextCompat.startForegroundService(context, intent);
}
{% endhighlight %}

既然是Service，那我们就看它的onHandleIntentInForeground方法。

{% highlight java %}
@Override 
protected void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
        CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
        return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer =
        new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);

    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
        heapDump.computeRetainedHeapSize);
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
{% endhighlight %}

首先会对传进来的intent进行判断，如果是null打印一条日志。如果intent不为null，那么我们就先获取intent传进来的两个参数监听类的名字LISTENER_CLASS_EXTRA和堆文件信息HEAPDUMP_EXTRA。接着就开始新建一个HeapAnalyzer来分析堆内存。然后HeapAnalyzer对象会调用```checkForLeak()```方法来检测是否内存泄漏的结果。最后将这个结果返回给我们的AbstractAnalysisResultService，用来进行显示。

不用多说，我们来看一下```checkForLeak()```方法。

{% highlight java %}
// HeapAnalyzer.java
public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey,
      boolean computeRetainedSize) {
    long analysisStartNanoTime = System.nanoTime();

    if (!heapDumpFile.exists()) {
        Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
        return failure(exception, since(analysisStartNanoTime));
    }

    try {
        listener.onProgressUpdate(READING_HEAP_DUMP_FILE);
        HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile); // 根据hprof文件，新建一个HprofBuffer对象
        HprofParser parser = new HprofParser(buffer); // 新建一个HprofParser解析器
        listener.onProgressUpdate(PARSING_HEAP_DUMP);
        Snapshot snapshot = parser.parse(); // 将hprof文件解析成Snapshot内存快照对象
        listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
        deduplicateGcRoots(snapshot); // 去重的GcRoots，对我们分析结果进行去重，把那些重复内存泄漏和它的GcRoots删除。
        listener.onProgressUpdate(FINDING_LEAKING_REF);
        Instance leakingRef = findLeakingReference(referenceKey, snapshot); // 根据referenceKey去查找内存快照中的泄露对象

        // False alarm, weak reference was cleared in between key check and heap dump.
        if (leakingRef == null) {
          return noLeak(since(analysisStartNanoTime)); // 没有发生泄漏
        }
        return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize); // 把整个泄露对象的路径找出来
    } catch (Throwable e) {
        return failure(e, since(analysisStartNanoTime));
    }
}
{% endhighlight %}

checkForLeak方法就是进一步分析内存，将我们前面创建好的hprof文件解析成内存快照Snapshot对象，就可以进行内存分析了。总而言之，checkForLeak方法做了三件事情：

1. 把hprof文件转为Snapshot
2. 优化GcRoots
3. 找出泄露的对象/找出泄露对象的最短路径

接下来我们会分析如何找出泄漏对象和泄露对象的最短路径

我们先看```findLeakingReference()```方法，它可以找到泄露对象。

{% highlight java %}
// HeapAnalyzer.java
private Instance findLeakingReference(String key, Snapshot snapshot) {
    ClassObj refClass = snapshot.findClass(KeyedWeakReference.class.getName()); // 由于我们给每个监测对象都创建了一个弱引用，这里我们可以通过弱引用来从snapshot中获取引用对象
    if (refClass == null) { // 如果引用对象为null，则报异常
        throw new IllegalStateException(
            "Could not find the " + KeyedWeakReference.class.getName() + " class in the heap dump.");
    }
    List<String> keysFound = new ArrayList<>();
    for (Instance instance : refClass.getInstancesList()) { // 通过不断遍历获取我们需要的引用对象的key值
        List<ClassInstance.FieldValue> values = classInstanceValues(instance);
        Object keyFieldValue = fieldValue(values, "key");
        if (keyFieldValue == null) {
            keysFound.add(null);
            continue;
        }
        String keyCandidate = asString(keyFieldValue);
        if (keyCandidate.equals(key)) { // 当key值相等时，说明找到了这个泄露变量的引用 
            return fieldValue(values, "referent");
        }
        keysFound.add(keyCandidate);
    }
    throw new IllegalStateException(
        "Could not find weak reference with key " + key + " in " + keysFound);
}
{% endhighlight %}

```findLeakingReference()```方法实际上做了三件事：

1. 在snapshot快照中找到第一个弱引用
2. 遍历这个对象的所有实例
3. 如果key值和最开始定义封装的key值相同，那么返回这个泄漏对象

我们再看```findLeakTrace()```方法，它可以找到泄露对象的最短路径。

{% highlight java %}
// HeapAnalyzer.java
private AnalysisResult findLeakTrace(long analysisStartNanoTime, Snapshot snapshot,
      Instance leakingRef, boolean computeRetainedSize) {

    listener.onProgressUpdate(FINDING_SHORTEST_PATH);
    ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
    ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef); // 这里找到了泄露对象的最短路径

    // False alarm, no strong reference path to GC Roots.
    if (result.leakingNode == null) {
        return noLeak(since(analysisStartNanoTime));
    }

    listener.onProgressUpdate(BUILDING_LEAK_TRACE);
    LeakTrace leakTrace = buildLeakTrace(result.leakingNode); // 展示在屏幕上的LeakTrace

    String className = leakingRef.getClassObj().getClassName();

    long retainedSize;
    if (computeRetainedSize) {

        listener.onProgressUpdate(COMPUTING_DOMINATORS);
        // Side effect: computes retained size.
        snapshot.computeDominators();

        Instance leakingInstance = result.leakingNode.instance;

        retainedSize = leakingInstance.getTotalRetainedSize(); // 用来计算泄露对象的占用内存大小

        // TODO: check O sources and see what happened to android.graphics.Bitmap.mBuffer
        if (SDK_INT <= N_MR1) {
            listener.onProgressUpdate(COMPUTING_BITMAP_SIZE);
            retainedSize += computeIgnoredBitmapRetainedSize(snapshot, leakingInstance);
        }
    } else {
      retainedSize = AnalysisResult.RETAINED_HEAP_SKIPPED;
    }

    return leakDetected(result.excludingKnownLeaks, className, leakTrace, retainedSize,
        since(analysisStartNanoTime));
}
{% endhighlight %}

### 总结

到这里我们对LeakCanary的分析就完成了，总结一下，LeakCanary一共做了三件事情：

1. 首先会创建一个refWatcher，启动一个ActivityRefWatcher，用来监听Activity的回收情况。
2. 通过ActivityLifecycleCallbacks把Activity的onDestroy生命周期关联。
3. 最后在线程池中去开始分析我们的内存泄漏。

