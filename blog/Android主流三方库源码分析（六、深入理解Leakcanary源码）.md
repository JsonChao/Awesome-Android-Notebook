---

		title:  Android主流三方库源码分析（六、深入理解Leakcanary源码）
		date: 2019/01/06 18:35:00   
		tags: 
		- Android主流三方库源码分析
		categories: 安卓主流三方库源码分析
		thumbnail: https://cdn.pixabay.com/photo/2016/08/15/06/05/jeju-island-cheonjiyeon-waterfall-1594589_960_720.jpg
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

在Android主流三方库源码分析系列的前几篇文章中，笔者已经对网络、图片、数据库、响应式编程中最热门的第三方开源框架进行了较为深入地讲解，如果有朋友对这四块感兴趣的话，可以去了解下。本篇，我将会对Android中的内存泄露检测框架Leakcanary的源码流程进行详细地讲解。

# 一、原理概述

首先，笔者仔细查看了Leakcanary官方的github仓库，最重要的便是对**Leakcanary是如何起作用的**（即原理）这一问题进行了阐述，我自己把它翻译成了易于理解的文字，主要分为如下7个步骤：

- 1、RefWatcher.watch()创建了一个KeyedWeakReference用于去观察对象。
- 2、然后，在后台线程中，它会检测引用是否被清除了，并且是否没有触发GC。
- 3、如果引用仍然没有被清除，那么它将会把堆栈信息保存在文件系统中的.hprof文件里。
- 4、HeapAnalyzerService被开启在一个独立的进程中，并且HeapAnalyzer使用了HAHA开源库解析了指定时刻的堆栈快照文件heap dump。
- 5、从heap dump中，HeapAnalyzer根据一个独特的引用key找到了KeyedWeakReference，并且定位了泄露的引用。
- 6、HeapAnalyzer为了确定是否有泄露，计算了到GC Roots的最短强引用路径，然后建立了导致泄露的链式引用。
- 7、这个结果被传回到app进程中的DisplayLeakService，然后一个泄露通知便展现出来了。

官方的原理简单来解释就是这样的：**在一个Activity执行完onDestroy()之后，将它放入WeakReference中，然后将这个WeakReference类型的Activity对象与ReferenceQueque关联。这时再从ReferenceQueque中查看是否有没有该对象，如果没有，执行gc，再次查看，还是没有的话则判断发生内存泄露了。最后用HAHA这个开源库去分析dump之后的heap内存。**

# 二、简单示例

下面这段是Leakcanary官方仓库的示例代码：

首先在你项目app下的build.gradle中配置:

    dependencies {
      debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.2'
      releaseImplementation   'com.squareup.leakcanary:leakcanary-android-no-op:1.6.2'
      // 可选，如果你使用支持库的fragments的话
      debugImplementation   'com.squareup.leakcanary:leakcanary-support-fragment:1.6.2'
    }
    
然后在你的Application中配置:

    public class WanAndroidApp extends Application {
    
        private RefWatcher refWatcher;
        
        public static RefWatcher getRefWatcher(Context context) {
            WanAndroidApp application = (WanAndroidApp)     context.getApplicationContext();
            return application.refWatcher;
        }
    
        @Override public void onCreate() {
          super.onCreate();
          if (LeakCanary.isInAnalyzerProcess(this)) {
            // 1
            return;
          }
          // 2
          refWatcher = LeakCanary.install(this);
        }
    }

在注释1处，会首先判断当前进程是否是Leakcanary专门用于分析heap内存的而创建的那个进程，即HeapAnalyzerService所在的进程，如果是的话，则不进行Application中的初始化功能。如果是当前应用所处的主进程的话，则会执行注释2处的LeakCanary.install(this)进行LeakCanary的安装。只需这样简单的几行代码，我们就可以在应用中检测是否产生了内存泄露了。当然，这样使用只会检测Activity和标准Fragment是否发生内存泄漏，如果要检测V4包的Fragment在执行完onDestroy()之后是否发生内存泄露的话，则需要在Fragment的onDestroy()方法中加上如下两行代码去监视当前的Fragment：

    RefWatcher refWatcher = WanAndroidApp.getRefWatcher(_mActivity);
    refWatcher.watch(this);

上面的**RefWatcher其实就是一个引用观察者对象，是用于监测当前实例对象的引用状态的**。从以上的分析可以了解到，核心代码就是LeakCanary.install(this)这行代码，接下来，就从这里出发将LeakCanary一步一步进行拆解。

# 三、源码分析

## 1、LeakCanary#install()

    public static @NonNull RefWatcher install(@NonNull Application application) {
      return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
          .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
          .buildAndInstall();
    }
    
在install()方法中的处理，可以分解为如下四步：

- 1、**refWatcher(application)**
- 2、**链式调用listenerServiceClass(DisplayLeakService.class)**
- 3、**链式调用excludedRefs(AndroidExcludedRefs.createAppDefaults().build())**
- 4、**链式调用buildAndInstall()**

首先，我们来看下第一步，这里调用了LeakCanary类的refWatcher方法，如下所示：

    public static @NonNull AndroidRefWatcherBuilder refWatcher(@NonNull Context context) {
      return new AndroidRefWatcherBuilder(context);
    }

然后新建了一个AndroidRefWatcherBuilder对象，再看看AndroidRefWatcherBuilder这个类。

## 2、AndroidRefWatcherBuilder

    /** A {@link RefWatcherBuilder} with appropriate Android defaults. */
    public final class AndroidRefWatcherBuilder extends RefWatcherBuilder<AndroidRefWatcherBuilder> {

    ...
    
        AndroidRefWatcherBuilder(@NonNull Context context) {
            this.context = context.getApplicationContext();
        }

    ...
    }

在AndroidRefWatcherBuilder的构造方法中仅仅是将外部传入的applicationContext对象保存起来了。**AndroidRefWatcherBuilder是一个适配Android平台的引用观察者构造器对象，它继承了RefWatcherBuilder，RefWatcherBuilder是一个负责建立引用观察者RefWatcher实例的基类构造器**。继续看看RefWatcherBuilder这个类。

## 3、RefWatcherBuilder

    public class RefWatcherBuilder<T extends RefWatcherBuilder<T>> {
    
        ...
        
        public RefWatcherBuilder() {
            heapDumpBuilder = new HeapDump.Builder();
        }
    
        ...
    }
    
在RefWatcher的基类构造器RefWatcherBuilder的构造方法中新建了一个HeapDump的构造器对象。其中**HeapDump就是一个保存heap dump信息的数据结构**。

接着来分析下install()方法中的链式调用的listenerServiceClass(DisplayLeakService.class)这部分逻辑。

## 4、AndroidRefWatcherBuilder#listenerServiceClass()

    public @NonNull AndroidRefWatcherBuilder listenerServiceClass(
      @NonNull Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
        return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
    }
    
在这里，传入了一个DisplayLeakService的Class对象，它的作用是展示泄露分析的结果日志，然后会展示一个用于跳转到显示泄露界面DisplayLeakActivity的通知。在listenerServiceClass()这个方法中新建了一个ServiceHeapDumpListener对象，下面看看它内部的操作。

## 5、ServiceHeapDumpListener

    public final class ServiceHeapDumpListener implements HeapDump.Listener {
    
        ...
        
        public ServiceHeapDumpListener(@NonNull final Context context,
            @NonNull final Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
          this.listenerServiceClass = checkNotNull(listenerServiceClass, "listenerServiceClass");
          this.context = checkNotNull(context, "context").getApplicationContext();
        }
        
        ...
    }

可以看到这里仅仅是在ServiceHeapDumpListener中保存了DisplayLeakService的Class对象和application对象。它的作用就是接收一个heap dump去分析。

然后我们继续看install()方法链式调用.excludedRefs(AndroidExcludedRefs.createAppDefaults().build())的这部分代码。先看AndroidExcludedRefs.createAppDefaults()。

## 6、AndroidExcludedRefs#createAppDefaults()

    public enum AndroidExcludedRefs {
    
        ...
    
        public static @NonNull ExcludedRefs.Builder createAppDefaults() {
          return createBuilder(EnumSet.allOf(AndroidExcludedRefs.class));
        }
        
        public static @NonNull ExcludedRefs.Builder createBuilder(EnumSet<AndroidExcludedRefs> refs) {
          ExcludedRefs.Builder excluded = ExcludedRefs.builder();
          for (AndroidExcludedRefs ref : refs) {
            if (ref.applies) {
              ref.add(excluded);
              ((ExcludedRefs.BuilderWithParams) excluded).named(ref.name());
            }
          }
          return excluded;
        }
        
        ...
    }
    
先来说下**AndroidExcludedRefs**这个类，它是一个enum类，它**声明了Android SDK和厂商定制的SDK中存在的内存泄露的case**，根据AndroidExcludedRefs这个类的类名就可看出这些case**都会被Leakcanary的监测过滤掉**。目前这个版本是有**46种**这样的**case**被包含在内，后续可能会一直增加。然后EnumSet.allOf(AndroidExcludedRefs.class)这个方法将会返回一个包含AndroidExcludedRefs元素类型的EnumSet。Enum是一个抽象类，在这里具体的实现类是**通用正规型的RegularEnumSet，如果Enum里面的元素个数大于64，则会使用存储大数据量的JumboEnumSet**。最后，在createBuilder这个方法里面构建了一个排除引用的建造器excluded，将各式各样的case分门别类地保存起来再返回出去。

最后，我们看到链式调用的最后一步buildAndInstall()。

## 7、AndroidRefWatcherBuilder#buildAndInstall()

    private boolean watchActivities = true;
    private boolean watchFragments = true;

    public @NonNull RefWatcher buildAndInstall() {
        // 1
        if (LeakCanaryInternals.installedRefWatcher != null) {
          throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
        }
        
        // 2
        RefWatcher refWatcher = build();
        if (refWatcher != DISABLED) {
          // 3
          LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
          if (watchActivities) {
            // 4
            ActivityRefWatcher.install(context, refWatcher);
          }
          if (watchFragments) {
            // 5
            FragmentRefWatcher.Helper.install(context, refWatcher);
          }
        }
        // 6
        LeakCanaryInternals.installedRefWatcher = refWatcher;
        return refWatcher;
    }
    
首先，在注释1处，会判断LeakCanaryInternals.installedRefWatcher是否已经被赋值，如果被赋值了，则会抛出异常，警告
buildAndInstall()这个方法应该仅仅只调用一次，在此方法结束时，即在注释6处，该LeakCanaryInternals.installedRefWatcher才会被赋值。再来看注释2处，调用了AndroidRefWatcherBuilder其基类RefWatcherBuilder的build()方法，我们它是如何建造的。

## 8、RefWatcherBuilder#build()

    public final RefWatcher build() {
        if (isDisabled()) {
          return RefWatcher.DISABLED;
        }
    
        if (heapDumpBuilder.excludedRefs == null) {
          heapDumpBuilder.excludedRefs(defaultExcludedRefs());
        }
    
        HeapDump.Listener heapDumpListener = this.heapDumpListener;
        if (heapDumpListener == null) {
          heapDumpListener = defaultHeapDumpListener();
        }
    
        DebuggerControl debuggerControl = this.debuggerControl;
        if (debuggerControl == null) {
          debuggerControl = defaultDebuggerControl();
        }
    
        HeapDumper heapDumper = this.heapDumper;
        if (heapDumper == null) {
          heapDumper = defaultHeapDumper();
        }
    
        WatchExecutor watchExecutor = this.watchExecutor;
        if (watchExecutor == null) {
          watchExecutor = defaultWatchExecutor();
        }
    
        GcTrigger gcTrigger = this.gcTrigger;
        if (gcTrigger == null) {
          gcTrigger = defaultGcTrigger();
        }
    
        if (heapDumpBuilder.reachabilityInspectorClasses == null) {
          heapDumpBuilder.reachabilityInspectorClasses(defa  ultReachabilityInspectorClasses());
        }
    
        return new RefWatcher(watchExecutor, debuggerControl, gcTrigger, heapDumper, heapDumpListener,
            heapDumpBuilder);
    }
    
可以看到，**RefWatcherBuilder包含了以下7个组成部分：**

- 1、**excludedRefs : 记录可以被忽略的泄漏路径**。

- 2、**heapDumpListener : 转储堆信息到hprof文件，并在解析完 hprof 文件后进行回调，最后通知 DisplayLeakService 弹出泄漏提醒**。

- 3、debuggerControl : 判断是否处于调试模式，调试模式中不会进行内存泄漏检测。为什么呢？因为**在调试过程中可能会保留上一个引用从而导致错误信息上报**。

- 4、**heapDumper : 堆信息转储者，负责dump 内存泄漏处的 heap 信息到 hprof 文件**。

- 5、**watchExecutor : 线程控制器，在 onDestroy() 之后并且在主线程空闲时执行内存泄漏检测**。

- 6、**gcTrigger : 用于 GC，watchExecutor 首次检测到可能的内存泄漏，会主动进行 GC，GC 之后会再检测一次，仍然泄漏的判定为内存泄漏，最后根据heapDump信息生成相应的泄漏引用链**。

- 7、**reachabilityInspectorClasses : 用于要进行可达性检测的类列表。**

最后，会使用建造者模式将这些组成部分构建成一个新的RefWatcher并将其返回。

我们继续看回到AndroidRefWatcherBuilder的注释3处的 LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true)这行代码。

## 9、LeakCanaryInternals#setEnabledAsync()

    public static void setEnabledAsync(Context context, final Class<?> componentClass,
    final boolean enabled) {
      final Context appContext = context.getApplicationContext();
      AsyncTask.THREAD_POOL_EXECUTOR.execute(new Runnable() {
        @Override public void run() {
          setEnabledBlocking(appContext, componentClass, enabled);
        }
      });
    }

在这里直接使用了**AsyncTask内部自带的THREAD_POOL_EXECUTOR线程池**进行阻塞式地显示DisplayLeakActivity。

然后我们再继续看AndroidRefWatcherBuilder的注释4处的代码。

## 10、ActivityRefWatcher#install()

    public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
        Application application = (Application) context.getApplicationContext();
        // 1
        ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
        
        // 2
        application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
    }

可以看到，在注释1处创建一个自己的activityRefWatcher实例，并在注释2处调用了application的registerActivityLifecycleCallbacks()方法，这样就能够监听activity对应的生命周期事件了。继续看看activityRefWatcher.lifecycleCallbacks里面的操作。

    private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
        new ActivityLifecycleCallbacksAdapter() {
          @Override public void onActivityDestroyed(Activity activity) {
              refWatcher.watch(activity);
          }
    };
    
    public abstract class ActivityLifecycleCallbacksAdapter
    implements Application.ActivityLifecycleCallbacks {
    
    }
    
很明显，这里**实现并重写了Application的ActivityLifecycleCallbacks的onActivityDestroyed()方法，这样便能在所有Activity执行完onDestroyed()方法之后调用 refWatcher.watch(activity)这行代码进行内存泄漏的检测了**。

我们再看到注释5处的FragmentRefWatcher.Helper.install(context, refWatcher)这行代码，

## 11、FragmentRefWatcher.Helper#install()

    public interface FragmentRefWatcher {

        void watchFragments(Activity activity);
    
        final class Helper {
        
          private static final String SUPPORT_FRAGMENT_REF_WATCHER_CLASS_NAME =
              "com.squareup.leakcanary.internal.SupportFragmentRefWatcher";
        
          public static void install(Context context, RefWatcher refWatcher) {
            List<FragmentRefWatcher> fragmentRefWatchers = new ArrayList<>();
        
            // 1
            if (SDK_INT >= O) {
              fragmentRefWatchers.add(new AndroidOFragmentRefWatcher(refWatcher));
            }
        
            // 2
            try {
              Class<?> fragmentRefWatcherClass = Class.forName(SUPPORT_FRAGMENT_REF_WATCHER_CLASS_NAME);
              Constructor<?> constructor =
                  fragmentRefWatcherClass.getDeclaredConstructor(RefWatcher.class);
              FragmentRefWatcher supportFragmentRefWatcher   =
                  (FragmentRefWatcher) constructor.newInstance(refWatcher);
              fragmentRefWatchers.add(supportFragmentRefWatcher);
            } catch (Exception ignored) {
            }
        
            if (fragmentRefWatchers.size() == 0) {
              return;
            }
        
            Helper helper = new Helper(fragmentRefWatchers);
        
            // 3
            Application application = (Application) context.getApplicationContext();
            application.registerActivityLifecycleCallbacks(helper.activityLifecycleCallbacks);
          }
          
        ...
    }
    
这里面的逻辑很简单，首先在注释1处将Android标准的Fragment的RefWatcher类，即AndroidOfFragmentRefWatcher添加到新创建的fragmentRefWatchers中。在注释2处**使用反射将leakcanary-support-fragment包下面的SupportFragmentRefWatcher添加进来，如果你在app的build.gradle下没有添加下面这行引用的话，则会拿不到此类，即LeakCanary只会检测Activity和标准Fragment这两种情况**。

    debugImplementation   'com.squareup.leakcanary:leakcanary-support-fragment:1.6.2'

继续看到注释3处helper.activityLifecycleCallbacks里面的代码。

    private final Application.ActivityLifecycleCallbacks activityLifecycleCallbacks =
        new ActivityLifecycleCallbacksAdapter() {
          @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            for (FragmentRefWatcher watcher : fragmentRefWatchers) {
                watcher.watchFragments(activity);
            }
        }
    };
        
可以看到，在Activity执行完onActivityCreated()方法之后，会调用指定watcher的watchFragments()方法，注意，这里的watcher可能有两种，但不管是哪一种，都会使用当前传入的activity获取到对应的FragmentManager/SupportFragmentManager对象，调用它的registerFragmentLifecycleCallbacks()方法，在对应的onDestroyView()和onDestoryed()方法执行完后，分别使用refWatcher.watch(view)和refWatcher.watch(fragment)进行内存泄漏的检测，代码如下所示。

    @Override public void onFragmentViewDestroyed(FragmentManager fm, Fragment fragment) {
        View view = fragment.getView();
        if (view != null) {
            refWatcher.watch(view);
        }
    }

    @Override
    public void onFragmentDestroyed(FragmentManagerfm, Fragment fragment) {
        refWatcher.watch(fragment);
    }
    
注意，下面到真正关键的地方了，接下来分析refWatcher.watch()这行代码。

#### 12、RefWatcher#watch()

    public void watch(Object watchedReference, String referenceName) {
        if (this == DISABLED) {
          return;
        }
        checkNotNull(watchedReference, "watchedReference");
        checkNotNull(referenceName, "referenceName");
        final long watchStartNanoTime = System.nanoTime();
        // 1
        String key = UUID.randomUUID().toString();
        // 2
        retainedKeys.add(key);
        // 3
        final KeyedWeakReference reference =
            new KeyedWeakReference(watchedReference, key, referenceName, queue);
    
        // 4
        ensureGoneAsync(watchStartNanoTime, reference);
    }
    
注意到在注释1处**使用随机的UUID保证了每个检测对象对应
key 的唯一性**。在注释2处将生成的key添加到类型为CopyOnWriteArraySet的Set集合中。在注释3处新建了一个自定义的弱引用KeyedWeakReference，看看它内部的实现。

## 13、KeyedWeakReference

    final class KeyedWeakReference extends WeakReference<Object> {
        public final String key;
        public final String name;
        
        KeyedWeakReference(Object referent, String key, String name,
            ReferenceQueue<Object> referenceQueue) {
          // 1
          super(checkNotNull(referent, "referent"), checkNotNull(referenceQueue, "referenceQueue"));
          this.key = checkNotNull(key, "key");
          this.name = checkNotNull(name, "name");
        }
    }
    
可以看到，**在KeyedWeakReference内部，使用了key和name标识了一个被检测的WeakReference对象**。在注释1处，**将弱引用和引用队列 ReferenceQueue 关联起来，如果弱引用reference持有的对象被GC回收，JVM就会把这个弱引用加入到与之关联的引用队列referenceQueue中。即 KeyedWeakReference 持有的 Activity 对象如果被GC回收，该对象就会加入到引用队列 referenceQueue 中**。

接着我们回到RefWatcher.watch()里注释4处的ensureGoneAsync()方法。

## 14、RefWatcher#ensureGoneAsync()

    private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
        // 1
        watchExecutor.execute(new Retryable() {
            @Override public Retryable.Result run() {
                // 2
                return ensureGone(reference watchStartNanoTime);
            }
        });
    }

在ensureGoneAsync()方法中，在注释1处使用 watchExecutor 执行了注释2处的 ensureGone 方法，watchExecutor 是 AndroidWatchExecutor 的实例。

下面看看watchExecutor内部的逻辑。

## 15、AndroidWatchExecutor

    public final class AndroidWatchExecutor implements WatchExecutor {

        ...
        
        public AndroidWatchExecutor(long initialDelayMillis)     {
          mainHandler = new Handler(Looper.getMainLooper());
          HandlerThread handlerThread = new HandlerThread(LEAK_CANARY_THREAD_NAME);
          handlerThread.start();
          // 1
          backgroundHandler = new Handler(handlerThread.getLooper());
          this.initialDelayMillis = initialDelayMillis;
          maxBackoffFactor = Long.MAX_VALUE / initialDelayMillis;
        }
        
        @Override public void execute(@NonNull Retryable retryable) {
          // 2
          if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
            waitForIdle(retryable, 0);
          } else {
            postWaitForIdle(retryable, 0);
          }
        }
        
        ...
    }

在注释1处**AndroidWatchExecutor的构造方法**中，注意到这里**使用HandlerThread的looper新建了一个backgroundHandler**，后面会用到。在注释2处，会判断当前线程是否是主线程，如果是，则直接调用waitForIdle()方法，如果不是，则调用postWaitForIdle()，来看看这个方法。

    private void postWaitForIdle(final Retryable retryable, final int failedAttempts) {
      mainHandler.post(new Runnable() {
        @Override public void run() {
          waitForIdle(retryable, failedAttempts);
        }
      });
    }
    
很清晰，这里使用了在构造方法中用主线程looper构造的mainHandler进行post，那么waitForIdle()最终也会在主线程执行。接着看看waitForIdle()的实现。

    private void waitForIdle(final Retryable retryable,     final int failedAttempts) {
      Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
        @Override public boolean queueIdle() {
          postToBackgroundWithDelay(retryable, failedAttempts);
          return false;
        }
      });
    }
    
这里**MessageQueue.IdleHandler()回调方法的作用是当 looper 空闲的时候，会回调 queueIdle 方法，利用这个机制我们可以实现第三方库的延迟初始化**，然后执行内部的postToBackgroundWithDelay()方法。接下来看看它的实现。

    private void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
      long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts),     maxBackoffFactor);
      // 1
      long delayMillis = initialDelayMillis * exponentialBackoffFactor;
      // 2
      backgroundHandler.postDelayed(new Runnable() {
        @Override public void run() {
          // 3
          Retryable.Result result = retryable.run();
          // 4
          if (result == RETRY) {
            postWaitForIdle(retryable, failedAttempts +   1);
          }
        }
      }, delayMillis);
    }
    
先看到注释4处，可以明白，postToBackgroundWithDelay()是一个递归方法，如果result 一直等于RETRY的话，则会一直执行postWaitForIdle()方法。在回到注释1处，这里initialDelayMillis 的默认值是 5s，因此delayMillis就是5s。在注释2处，使用了在构造方法中用HandlerThread的looper新建的backgroundHandler进行异步延时执行retryable的run()方法。这个run()方法里执行的就是RefWatcher的ensureGoneAsync()方法中注释2处的ensureGone()这行代码，继续看它内部的逻辑。

## 16、RefWatcher#ensureGone()

    Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
        long gcStartNanoTime = System.nanoTime();
        long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime -     watchStartNanoTime);
    
        // 1
        removeWeaklyReachableReferences();
    
        // 2
        if (debuggerControl.isDebuggerAttached()) {
          // The debugger can create false leaks.
          return RETRY;
        }
        
        // 3
        if (gone(reference)) {
          return DONE;
        }
        
        // 4
        gcTrigger.runGc();
        removeWeaklyReachableReferences();
        
        // 5
        if (!gone(reference)) {
          long startDumpHeap = System.nanoTime();
          long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
    
          File heapDumpFile = heapDumper.dumpHeap();
          if (heapDumpFile == RETRY_LATER) {
            // Could not dump the heap.
            return RETRY;
          }
          
          long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
    
          HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
              .referenceName(reference.name)
              .watchDurationMs(watchDurationMs)
              .gcDurationMs(gcDurationMs)
              .heapDumpDurationMs(heapDumpDurationMs)
              .build();
    
          heapdumpListener.analyze(heapDump);
        }
        return DONE;
    }
    
在注释1处，执行了removeWeaklyReachableReferences()这个方法，接下来分析下它的含义。

    private void removeWeaklyReachableReferences() {
        KeyedWeakReference ref;
        while ((ref = (KeyedWeakReference) queue.poll()) != null) {
            retainedKeys.remove(ref.key);
        }
    }
    
这里使用了while循环遍历 ReferenceQueue ，并从 retainedKeys中移除对应的Reference。

再看到注释2处，**当Android设备处于debug状态时，会直接返回RETRY进行延时重试检测的操作**。在注释3处，我们看看gone(reference)这个方法的逻辑。

    private boolean gone(KeyedWeakReference reference) {
        return !retainedKeys.contains(reference.key);
    }
    
这里会**判断 retainedKeys 集合中是否还含有 reference，若没有，证明已经被回收了，若含有，可能已经发生内存泄露（或Gc还没有执行回收）**。前面的分析中我们知道了 **reference 被回收的时候，会被加进 referenceQueue 里面，然后我们会调用removeWeaklyReachableReferences()遍历 referenceQueue 移除掉 retainedKeys 里面的 refrence**。

接着我们看到注释4处，执行了gcTrigger的runGc()方法进行垃圾回收，然后使用了removeWeaklyReachableReferences()方法移除已经被回收的引用。这里我们再深入地分析下runGc()的实现。

    GcTrigger DEFAULT = new GcTrigger() {
        @Override public void runGc() {
          // Code taken from AOSP FinalizationTest:
          // https://android.googlesource.com/platform/libc  ore/+/master/support/src/test/java/libcore/
          // java/lang/ref/FinalizationTester.java
          // System.gc() does not garbage collect every   time. Runtime.gc() is
          // more likely to perform a gc.
          Runtime.getRuntime().gc();
          enqueueReferences();
          System.runFinalization();
        }
    
        private void enqueueReferences() {
          // Hack. We don't have a programmatic way to wait   for the reference queue daemon to move
          // references to the appropriate queues.
          try {
            Thread.sleep(100);
          } catch (InterruptedException e) {
            throw new AssertionError();
          }
        }
    };

这里并没有使用System.gc()方法进行回收，因为**system.gc()并不会每次都执行**。而是**从AOSP中拷贝一段GC回收的代码，从而相比System.gc()更能够保证垃圾回收的工作**。

最后我们分析下注释5处的代码处理。首先会判断activity是否被回收，如果还没有被回收，则证明发生内存泄露，进行if判断里面的操作。在里面先调用堆信息转储者heapDumper的dumpHeap()生成相应的 hprof 文件。这里的heapDumper是一个HeapDumper接口，具体的实现是AndroidHeapDumper。我们分析下AndroidHeapDumper的dumpHeap()方法是如何生成hprof文件的。

    public File dumpHeap() {
        File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();

        if (heapDumpFile == RETRY_LATER) {
            return RETRY_LATER;
        }

        ...
        
        try {
          Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
          ...
          
          return heapDumpFile;
        } catch (Exception e) {
          ...
          // Abort heap dump
          return RETRY_LATER;
        }
    }
    
    
这里的核心操作就是**调用了Android SDK的API Debug.dumpHprofData() 来生成 hprof 文件**。

如果这个文件等于RETRY_LATER则表示生成失败，直接返回RETRY进行延时重试检测的操作。如果不等于的话，则表示生成成功，最后会**执行heapdumpListener的analyze()对新创建的HeapDump对象进行泄漏分析**。由前面对AndroidRefWatcherBuilder的listenerServiceClass()的分析可知，heapdumpListener的实现
就是ServiceHeapDumpListener，接着看到ServiceHeapDumpListener的analyze方法。

## 17、ServiceHeapDumpListener#analyze()

    @Override public void analyze(@NonNull HeapDump heapDump) {
        checkNotNull(heapDump, "heapDump");
        HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
    }
        
可以看到，这里**执行了HeapAnalyzerService的runAnalysis()方法，为了避免降低app进程的性能或占用内存，这里将HeapAnalyzerService设置在了一个独立的进程中**。接着继续分析runAnalysis()方法里面的处理。

    public final class HeapAnalyzerService extends ForegroundService
    implements AnalyzerProgressListener {
    
        ...

        public static void runAnalysis(Context context, HeapDump heapDump,
        Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
            ...
            
            ContextCompat.startForegroundService(context, intent);
        }
    
        ...
        
        @Override protected void onHandleIntentInForeground(@Nullable Intent intent) {
            ...

            // 1
            HeapAnalyzer heapAnalyzer =
                new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);

            // 2
            AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
            heapDump.computeRetainedHeapSize);
            
            // 3
            AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
        }
            ...
    }
    
这里的HeapAnalyzerService实质是一个类型为IntentService的ForegroundService，执行startForegroundService()之后，会回调onHandleIntentInForeground()方法。注释1处，首先会新建一个**HeapAnalyzer**对象，顾名思义，它就是**根据RefWatcher生成的heap dumps信息来分析被怀疑的泄漏是否是真的**。在注释2处，然后会**调用它的checkForLeak()方法去使用haha库解析 hprof文件**，如下所示：

    public @NonNull AnalysisResult checkForLeak(@NonNull File heapDumpFile,
      @NonNull String referenceKey,
      boolean computeRetainedSize) {
        ...
        
        try {
        listener.onProgressUpdate(READING_HEAP_DUMP_FILE);
        // 1
        HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
      
        // 2
        HprofParser parser = new HprofParser(buffer);
        listener.onProgressUpdate(PARSING_HEAP_DUMP);
        Snapshot snapshot = parser.parse();
      
        listener.onProgressUpdate(DEDUPLICATING_GC_ROOTS);
        // 3
        deduplicateGcRoots(snapshot);
        listener.onProgressUpdate(FINDING_LEAKING_REF);
      
        // 4
        Instance leakingRef = findLeakingReference(referenceKey, snapshot);

        // 5
        if (leakingRef == null) {
            return noLeak(since(analysisStartNanoTime));
        }
      
        // 6
        return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, computeRetainedSize);
        } catch (Throwable e) {
        return failure(e, since(analysisStartNanoTime));
        }
    }
    
在注释1处，会新建一个**内存映射缓存文件buffer**。在注释2处，会**使用buffer新建一个HprofParser解析器去解析出对应的引用内存快照文件snapshot**。在注释3处，**为了减少在Android 6.0版本中重复GCRoots带来的内存压力的影响，使用deduplicateGcRoots()删除了gcRoots中重复的根对象RootObj**。在注释4处，**调用了findLeakingReference()方法将传入的referenceKey和snapshot对象里面所有类实例的字段值对应的keyCandidate进行比较，如果没有相等的，则表示没有发生内存泄漏**，直接调用注释5处的代码返回一个没有泄漏的分析结果AnalysisResult对象。**如果找到了相等的，则表示发生了内存泄漏**，执行注释6处的代码findLeakTrace()方法返回一个有泄漏分析结果的AnalysisResult对象。

最后，我们来分析下HeapAnalyzerService中注释3处的AbstractAnalysisResultService.sendResultToListener()方法，很明显，这里AbstractAnalysisResultService的实现类就是我们刚开始分析的用于展示泄漏路径信息的DisplayLeakService对象。在里面直接**创建一个由PendingIntent构建的泄漏通知用于供用户点击去展示详细的泄漏界面DisplayLeakActivity**。核心代码如下所示：

    public class DisplayLeakService extends AbstractAnalysisResultService {

        @Override
        protected final void onHeapAnalyzed(@NonNull AnalyzedHeap analyzedHeap) {
        
            ...
            
            boolean resultSaved = false;
            boolean shouldSaveResult = result.leakFound || result.failure != null;
            if (shouldSaveResult) {
                heapDump = renameHeapdump(heapDump);
                // 1
                resultSaved = saveResult(heapDump, result);
            }
            
            if (!shouldSaveResult) {
                ...
                showNotification(null, contentTitle, contentText);
            } else if (resultSaved) {
                ...
                // 2
                PendingIntent pendingIntent =
                    DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);
    
                ...
                
                showNotification(pendingIntent, contentTitle, contentText);
            } else {
                 onAnalysisResultFailure(getString(R.string.leak_canary_could_not_save_text));
            }
        
        ...
    }
    
    @Override protected final void onAnalysisResultFailure(String failureMessage) {
        super.onAnalysisResultFailure(failureMessage);
        String failureTitle = getString(R.string.leak_canary_result_failure_title);
        showNotification(null, failureTitle, failureMessage);
    }
    
可以看到，只要当分析的堆信息文件保存成功之后，即在注释1处返回的resultSaved为true时，才会执行注释2处的逻辑，即创建一个供用户点击跳转到DisplayLeakActivity的延时通知。最后给出一张源码流程图用于回顾本篇文章中LeakCanary的运作流程：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0363331db824e7ab342e4bd74702a93~tplv-k3u1fbpfcp-zoom-1.image)

### 四、总结

性能优化一直是Android中进阶和深入的方向之一，而内存泄漏一直是性能优化中比较重要的一部分，Android Studio自身提供了MAT等工具去分析内存泄漏，但是分析起来比较耗时耗力，因而才诞生了LeakCanary，它的使用非常简单，但是经过对它的深入分析之后，才发现，**简单的API后面往往藏着许多复杂的逻辑处理，尝试去领悟它们，你可能会发现不一样的世界**。


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6b4b4edabb6e4ca285fcdc59f99a0dd9~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、LeakCanary V1.6.2 源码

2、[一步步拆解 LeakCanary](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650243402&idx=1&sn=e7632788e8e147320b26a1b006cabf4c&chksm=88637025bf14f933dcf90fbd1d7f9090e3802dd37759cac014ac1ed73c455fb9b5d9953bb89f&scene=38#wechat_redirect)

3、[深入理解 Android 之 LeakCanary 源码解析](https://allenwu.itscoder.com/leakcanary-source)


## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e88dbbb593f6456db55efc5c7a1d2e0a~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。