---

		title:  深入探索Android启动速度优化（上）
		date: 2020/2/11 22:02:00   
		tags: 
		- 性能优化
		categories: 性能优化
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


在性能优化的整个知识体系中，最重要的就是稳定性优化，在上一篇文章 [《深入探索Android稳定性优化》](https://juejin.im/post/6844903972587716621) 中我们已经深入探索了Android稳定性优化的疆域。那么，**除了稳定性以外，对于性能纬度来说，哪个方面的性能是最重要的呢**？毫无疑问，就是**应用的启动速度**。下面，**就让我们扬起航帆，一起来逐步深入探索Android启动速度优化的奥秘**。


# 思维导图大纲

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ba37d61c8ae439fb27cd4723a576816~tplv-k3u1fbpfcp-zoom-1.image)


# 目录

- 一、[启动优化的意义](https://juejin.im/post/6844904093786308622#heading-4)
- 二、[应用启动流程](https://juejin.im/post/6844904093786308622#heading-5)
    - 1 、应用启动的类型
    - 2、冷启动分析及其优化方向
- 三、[启动耗时检测](https://juejin.im/post/6844904093786308622#heading-21)
    - 1、查看Logcat
    - 2、adb shell
    - 3、代码打点（函数插桩）
    - 4、AOP(Aspect Oriented Programming) 打点
    - 5、启动速度分析工具 — TraceView
    - 6、启动速度分析工具 — Systrace
    - 7、启动监控
- 四、[启动优化常规方案](https://juejin.im/post/6844904093786308622#heading-88)
    - 1、主题切换
    - 2、第三方库懒加载
    - 3、异步初始化预备知识-线程优化
    - 4、异步初始化
    - 5、延迟初始化
    - 6、Multidex预加载优化
    - 7、类预加载优化
    - 8、WebView启动优化
    - 9、页面数据预加载
    - 10、启动阶段不启动子进程
    - 11、闪屏页与主页的绘制优化


# 一、启动优化的意义

如果我们去一家餐厅吃饭，在点餐的时候等了半天都没有服务人员过来，可能就没有耐心等待直接走了。

对于App来说，也是同样如此，**如果用户点击App后，App半天都打不开，用户就可能失去耐心卸载应用**。

**启动速度是用户对我们App的第一体验**，打开应用后才能去使用其中提供的强大功能，就算我们应用的内部界面设计的再精美，功能再强大，**如果启动速度过慢，用户第一印象就会很差**。

因此，拯救App的启动速度，迫在眉睫。


# 二、应用启动流程

## 1 、应用启动的类型

应用启动的类型总共分为如下三种：

- **冷启动**
- **热启动**
- **温启动**


下面，我们来详细分析下各个启动类型的特点及流程。


### 冷启动

从点击应用图标到UI界面完全显示且用户可操作的全部过程。


#### 特点

耗时最多，**衡量标准**。


#### 启动流程

Click Event    ->     IPC     ->     Process.start     ->      ActivityThread ->    bindApplication      ->      LifeCycle    ->     ViewRootImpl


首先，用户进行了一个点击操作，这个点击事件它会触发一个IPC的操作，之后便会执行到Process的start方法中，这个方法是用于进程创建的，接着，便会执行到ActivityThread的main方法，这个方法可以看做是我们单个App进程的入口，相当于Java进程的main方法，在其中会**执行消息循环的创建与主线程Handler的创建**，创建完成之后，就会执行到 bindApplication 方法，在这里**使用了反射去创建 Application**以及调用了 Application相关的生命周期，Application结束之后，便会执行Activity的生命周期，在Activity生命周期结束之后，**最后，就会执行到 ViewRootImpl，这时才会进行真正的一个页面的绘制**。


### 热启动

直接从后台切换到前台。


#### 特点

启动速度最快。


### 温启动

只会重走Activity的生命周期，而不会重走进程的创建，Application的创建与生命周期等。


#### 特点

较快，介于冷启动和热启动之间的一个速度。


#### 启动流程

LifeCycle    ->     ViewRootImpl


#### ViewRootImpl是什么？

它是GUI管理系统与GUI呈现系统之间的桥梁。**每一个ViewRootImpl关联一个Window，
ViewRootImpl 最终会通过它的setView方法绑定Window所对应的View，并通过其performTraversals方法对View进行布局、测量和绘制**。


## 2、冷启动分析及其优化方向

### 冷启动涉及的相关任务

#### 冷启动之前

- 首先，会启动App
- 然后，加载空白Window
- 最后，创建进程


需要注意的是，这些都是系统的行为，一般情况下我们是无法直接干预的。


#### 随后任务

- 首先，创建Application
- 启动主线程
- 创建MainActivity
- 加载布局
- 布置屏幕
- 首帧绘制


通常到了**界面首帧绘制完成**后，我们就可以认为**启动已经结束**了。


### 优化方向

我们的优化方向就是 **Application和Activity的生命周期** 这个阶段，因为这个阶段的时机对于我们来说是**可控的**。


# 三、启动耗时检测

## 1、查看Logcat

在Android Studio Logcat中**过滤关键字“Displayed”**，可以看到对应的冷启动耗时日志。


## 2、adb shell

使用adb shell获取应用的启动时间

    // 其中的AppstartActivity全路径可以省略前面的packageName
    adb shell am start -W [packageName]/[AppstartActivity全路径]


执行后会得到三个时间：ThisTime、TotalTime和WaitTime，详情如下：

### ThisTime

表示最后一个Activity启动耗时。


### TotalTime

表示所有Activity启动耗时。


### WaitTime

表示AMS启动Activity的总耗时。


一般来说，只需查看得到的TotalTime，即应用的启动时间，其包括 **创建进程 + Application初始化 + Activity初始化到界面显示** 的过程。

### 特点：

- 1、**线下使用方便，不能带到线上**。
- 2、**非严谨、精确时间**。


## 3、代码打点（函数插桩）

可以写一个统计耗时的工具类来记录整个过程的耗时情况。其中需要注意的有：

- 在上传数据到服务器时**建议根据用户ID的尾号来抽样上报**。
- 在项目中**核心基类的关键回调函数和核心方法**中加入打点。

其代码如下所示：


    /**
    * 耗时监视器对象，记录整个过程的耗时情况，可以用在很多需要统计的地方，比如Activity的启动耗时和Fragment的启动耗时。
    */
    public class TimeMonitor {
    
        private final String TAG = TimeMonitor.class.getSimpleName();
        private int mMonitord = -1;
        
        // 保存一个耗时统计模块的各种耗时，tag对应某一个阶段的时间
        private HashMap<String, Long> mTimeTag = new HashMap<>();
        private long mStartTime = 0;

        public TimeMonitor(int mMonitorId) {
            Log.d(TAG, "init TimeMonitor id: " + mMonitorId);
            this.mMonitorId = mMonitorId;
        }

        public int getMonitorId() {
            return mMonitorId;
        }

        public void startMonitor() {
            // 每次重新启动都把前面的数据清除，避免统计错误的数据
            if (mTimeTag.size() > 0) {
            mTimeTag.clear();
            }
            mStartTime = System.currentTimeMillis();
        }

        /**
        * 每打一次点，记录某个tag的耗时
        */
        public void recordingTimeTag(String tag) {
            // 若保存过相同的tag，先清除
            if (mTimeTag.get(tag) != null) {
                mTimeTag.remove(tag);
            }
            long time = System.currentTimeMillis() - mStartTime;
            Log.d(TAG, tag + ": " + time);
            mTimeTag.put(tag, time);
        }

        public void end(String tag, boolean writeLog) {
            recordingTimeTag(tag);
            end(writeLog);
        }

        public void end(boolean writeLog) {
            if (writeLog) {
                //写入到本地文件
            }
        }

        public HashMap<String, Long> getTimeTags() {
            return mTimeTag;
        }
    }


为了**使代码更好管理**，我们需要定义一个**打点配置类**，如下所示：


    /**
    * 打点配置类，用于统计各阶段的耗时，便于代码的维护和管理。
    */
    public final class TimeMonitorConfig {
    
        // 应用启动耗时
        public static final int TIME_MONITOR_ID_APPLICATION_START = 1;
    }


此外，耗时统计可能会在多个模块和类中需要打点，所以需要一个**单例类来管理各个耗时统计的数据**：


    /**
    * 采用单例管理各个耗时统计的数据。
    */
    public class TimeMonitorManager {
    
        private static TimeMonitorManager mTimeMonitorManager = null;
    private HashMap<Integer, TimeMonitor> mTimeMonitorMap = null;

        public synchronized static TimeMonitorManager getInstance() {
            if (mTimeMonitorManager == null) {
                mTimeMonitorManager = new TimeMonitorManager();
            }
            return mTimeMonitorManager;
        }

        public TimeMonitorManager() {
            this.mTimeMonitorMap = new HashMap<Integer, TimeMonitor>();
        }

        /**
         * 初始化打点模块
        */
        public void resetTimeMonitor(int id) {
            if (mTimeMonitorMap.get(id) != null) {
                mTimeMonitorMap.remove(id);
            }
            getTimeMonitor(id).startMonitor();
        }

        /**
        * 获取打点器
        */
        public TimeMonitor getTimeMonitor(int id) {
            TimeMonitor monitor = mTimeMonitorMap.get(id);
            if (monitor == null) {
                monitor = new TimeMonitor(id);
                mTimeMonitorMap.put(id, monitor);
            }
            return monitor;
        }
    }
    
    
主要在以下几个方面需要打点：

- **应用程序的生命周期节点**。
- **启动时需要初始化的重要方法**，例如数据库初始化，读取本地的一些数据。
- **其他耗时的一些算法**。


例如，启动时在Application和第一个Activity加入打点统计：


### Application 打点


    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        TimeMonitorManager.getInstance().resetTimeMonitor(TimeMonitorConfig.TIME_MONITOR_ID_APPLICATION_START);
    }
  
    @Override
    public void onCreate() {
        super.onCreate();
        SoLoader.init(this, /* native exopackage */ false);
        TimeMonitorManager.getInstance().getTimeMonitor(TimeMonitorConfig.TIME_MONITOR_ID_APPLICATION_START).recordingTimeTag("Application-onCreate");
    }


### 第一个Activity打点


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        TimeMonitorManager.getInstance().getTimeMonitor(TimeMonitorConfig.TIME_MONITOR_ID_APPLICATION_START).recordingTimeTag("SplashActivity-onCreate");
        super.onCreate(savedInstanceState);
        
        initData();
        
        TimeMonitorManager.getInstance().getTimeMonitor(TimeMonitorConfig.TIME_MONITOR_ID_APPLICATION_START).recordingTimeTag("SplashActivity-onCreate-Over");
    }

    @Override
    protected void onStart() {
        super.onStart();
        TimeMonitorManager.getInstance().getTimeMonitor(TimeMonitorConfig.TIME_MONITOR_ID_APPLICATION_START).end("SplashActivity-onStart", false);
    }


### 特点

**精确，可带到线上，但是代码有侵入性，修改成本高**。


### 注意事项

- 1、在上传数据到服务器时**建议根据用户ID的尾号来抽样上报**。
- 2、**onWindowFocusChanged只是首帧时间，App启动完成的结束点应该是真实数据展示出来的时候（通常来说都是首帧数据），如列表第一条数据展示，记得使用getViewTreeObserver().addOnPreDrawListener()（在API 16以上可以使用addOnDrawListener），它会把任务延迟到列表显示后再执行**，例如，在[Awesome-WanAndroid]()项目的主页就有一个RecyclerView实现的列表，启动结束的时间就是列表的首帧时间，也即列表第一条数据展示的时候。这里，我们直接在RecyclerView的适配器ArticleListAdapter的convert（onBindViewHolder）方法中加上如下代码即可：
    
    
        if (helper.getLayoutPosition() == 1 && !mHasRecorded) {
            mHasRecorded = true;
            helper.getView(R.id.item_search_pager_group).getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
                @Override
                public boolean onPreDraw() {
                    helper.getView(R.id.item_search_pager_group).getViewTreeObserver().removeOnPreDrawListener(this);
                    LogHelper.i("FeedShow");
                    return true;
                }
            });
        }


具体的实例代码可在 [这里查看](https://github.com/JsonChao/Awesome-WanAndroid/blob/master/app/src/main/java/json/chao/com/wanandroid/ui/mainpager/adapter/ArticleListAdapter.java)。

#### 为什么不使用onWindowFocusChanged这个方法作为启动结束点？

因为用户看到真实的界面是需要有网络请求返回真实数据的，但是onWindowFocusChanged只是界面绘制的首帧时机，但是列表中的数据是需要从网络中下载得到的，所以应该以列表的首帧数据作为启动结束点。


## 4、AOP(Aspect Oriented Programming) 打点

面向切面编程，**通过预编译和运行期动态代理实现程序功能统一维护**的一种技术。


### 1、作用

利用AOP可**以对业务逻辑的各个部分进行隔离**，从而使得业务逻辑各部分之间的**耦合性降低**，**提高程序的可重用性**，同时大大**提高了开发效率**。


### 2、AOP核心概念

#### 1、横切关注点

对哪些方法进行拦截，拦截后怎么处理。


#### 2、切面（Aspect）

类是对物体特征的抽象，**切面就是对横切关注点的抽象**。


#### 3、连接点（JoinPoint）

被拦截到的点（方法、字段、构造器）。


#### 4、切入点（PointCut）

对JoinPoint进行拦截的定义。


#### 5、通知（Advice）

**拦截到JoinPoint后要执行的代码**，分为**前置、后置、环绕**三种类型。


### 3、准备：接入AspectJx进行切面编码


首先，为了在Android使用AOP埋点需要引入AspectJ，在项目根目录的build.gradle下加入：


    classpath 'com.hujiang.aspectjx:gradle-android-plugin- aspectjx:2.0.0'


然后，在app目录下的build.gradle下加入：


    apply plugin: 'android-aspectjx'
    implement 'org.aspectj:aspectjrt:1.8.+'
  
    
### 4、AOP埋点实战

JoinPoint一般定位在如下位置

- 1、**函数调用**
- 2、**获取、设置变量**
- 3、**类初始化**


**使用PointCut对我们指定的连接点进行拦截，通过Advice，就可以拦截到JoinPoint后要执行的代码**。Advice通常有以下几种类型：

- 1、**Before**：PointCut之前执行
- 2、**After**：PointCut之后执行
- 3、**Around**：PointCut之前、之后分别执行


首先，我们举一个小栗子：


    @Before("execution(* android.app.Activity.on**(..))")
    public void onActivityCalled(JoinPoint joinPoint) throws Throwable {
    ...
    }
    
    
**在 execution 中的是一个匹配规则**，第一个 * 代表**匹配任意的方法返回值**，后面的语法代码**匹配所有Activity中on开头的方法**。

其中execution是处理Join Point的类型，在AspectJx中共有两种类型，如下所示：

- 1、**call**：插入在函数体里面
- 2、**execution**：插入在函数体外面


#### 如何统计Application中的所有方法耗时？


    @Aspect
    public class ApplicationAop {
    
        @Around("call (* com.json.chao.application.BaseApplication.**(..))")
        public void getTime(ProceedingJoinPoint joinPoint) {
        Signature signature = joinPoint.getSignature();
        String name = signature.toShortString();
        long time = System.currentTimeMillis();
        try {
            joinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        Log.i(TAG, name + " cost" +     (System.currentTimeMillis() - time));
        }
    }
    
    
在上述代码中，我们需要注意 **不同的Action类型其对应的方法入参是不同的**，具体的差异如下所示：

- 当Action为**Before、After**时，方法入参为**JoinPoint**。
- 当Action为**Around**时，方法入参为**ProceedingPoint**。


##### Around和Before、After的最大区别:

**ProceedingPoint**不同于JoinPoint，其**提供了proceed方法执行目标方法**。


### 5、总结AOP特性

- 1、**无侵入性**
- 2、**修改方便，建议使用**


## 4、启动速度分析工具 — TraceView

### 1、使用方式

- 1、代码中添加：**Debug.startMethodTracing()、检测方法、Debug.stopMethodTracing()**。（需要**使用adb pull将生成的**.trace文件导出到电脑，然后使用Android Studio的Profiler进行加载）
- 2、打开 **Profiler  ->  CPU   ->    点击 Record   ->  点击 Stop  ->  查看Profiler下方Top Down/Bottom Up 区域**，以找出**耗时的热点方法**。


### 2、Profile CPU

使用 Profile 的 CPU 模块可以帮我们快速找到耗时的热点方法，下面，我们来详细来分析一下这个模块。


### 1、Trace types

Trace types 有四种，如下所示。

#### 1、Trace Java Methods

会记录每个方法的时间、CPU信息。对运行时性能影响较大。


#### 2、Sample Java Methods

相比于Trace Java Methods会记录每个方法的时间、CPU信息，它**会在应用的Java代码执行期间频繁捕获应用的调用堆栈**，对运行时性能的影响比较小，**能够记录更大的数据区域**。


#### 3、Sample C/C++ Functions

需部署到**Android 8.0及以上**设备，**内部使用simpleperf跟踪应用的native代码**，也可以命令行使用simpleperf。


#### 4、Trace System Calls

- 检查**应用与系统资源的交互情况**。
- 查看所有**核心的CPU瓶颈**。
- **内部采用systrace**，也可以使用systrace命令。


### 2、Event timeline

用于显示**应用程序在其生命周期中转换不同状态的活动**，如用户交互、屏幕旋转事件等。


### 3、CPU timeline

用于显示应用程序 **实时CPU使用率、其它进程实时CPU使用率、应用程序使用的线程总数**。


### 4、Thread activity timeline

**列出应用程序进程中的每个线程**，并**使用了不同的颜色在其时间轴上指示其活动**。

- **绿色**：线程处于**活动状态**或**准备好使用CPU**。
- **黄色**：线程正**等待IO**操作。（重要）
- **灰色**：线程正在**睡眠**，**不消耗CPU时间**。


### 5、检查跟踪数据窗口

Profile提供的检查跟踪数据窗口有四种，如下所示：

#### 1、Call Chart

提供函数跟踪数据的图形表示形式。

- **水平轴**：表示**调用的时间段和时间**。
- **垂直轴**：显示**被调用方**。
- **橙色**：**系统API**。
- **绿色**：**应用自有方法**。
- **蓝色**：**第三方API**（包括**Java API**）。


##### 提示

右键点击 Jump to source 跳转至指定函数。


#### 2、Flame Chart

**将具有相同调用方顺序的完全相同的方法收集起来**。

- **水平轴**：**执行每个方法的相对时间量**。
- **垂直轴**：显示**被调用方**。


##### 使用技巧

看**顶层**的**哪个函数占据的宽度最大（表现为平顶）**，可能**存在性能问题**。


#### 3、Top Down

- **递归调用列表**，提供**self、children、total时间和比率来表示被调用的函数信息**。
- **Flame Chart是Top Down列表数据的图形化**。


#### 4、Bottom Up

- **展开函数会显示其调用方**。
- **按照消耗CPU时间由多到少的顺序对函数排序**。


#### 注意事项

我们在查看上面4个跟踪数据的区域时，应该注意右侧的两个时间，如下所示：

- **Wall Clock Time**：**程序执行时间**。
- **Thread Time**：**CPU执行的时间**。


### 3、TraceView小结

#### 特点

- 1、**图形的形式展示执行时间、调用栈**等。
- 2、**信息全面，包含所有线程**。
- 3、**运行时开销严重，整体都会变慢**，得出的**结果并不真实**。
- 4、**找到最耗费时间的路径：Flame Chart、Top Down**。
- 5、**找到最耗费时间的节点：Bottom Up**。


#### 作用

主要做热点分析，用来得到以下两种数据：

- **单次执行最耗时的方法**。
- **执行次数最多的方法**。


## 5、启动速度分析工具 — Systrace

### 1、使用方式：代码插桩

首先，我们可以定义一个Trace静态工厂类，将Trace.begainSection()，Trace.endSection()封装成i、o方法，然后再在想要分析的方法前后进行插桩即可。

然后，在命令行下执行systrace.py脚本，命令如下所示：

    python /Users/quchao/Library/Android/sdk/platform-tools/systrace/systrace.py -t 20 sched gfx view wm am app webview -a "com.wanandroid.json.chao" -o ~/Documents/open-project/systrace_data/wanandroid_start_1.html


具体参数含义如下：

- -t：指定统计时间为20s。
- shced：cpu调度信息。
- gfx：图形信息。
- view：视图。
- wm：窗口管理。
- am：活动管理。
- app：应用信息。
- webview：webview信息。
- -a：指定目标应用程序的包名。
- -o：生成的systrace.html文件。
 

#### 如何查看数据？

在**UIThread**一栏可以看到**核心的系统方法时间区域**和我们自己使用**代码插桩捕获的方法时间区域**。

### 2、Systrace原理

- 首先，**在系统的一些关键链路（如SystemServcie、虚拟机、Binder驱动）插入一些信息（Label）**。
- 然后，**通过Label的开始和结束来确定某个核心过程的执行时间，并把这些Label信息收集起来得到系统关键路径的运行时间信息，最后得到整个系统的运行性能信息**;

其中，Android Framework 里面一些重要的模块都插入了label信息，用户App中也可以添加自定义的Lable。


### 3、Systrace小结

#### 特点

- 结合**Android内核的数据**，生成**Html**报告。
- **系统版本越高，Android Framework中添加的系统可用Label就越多，能够支持和分析的系统模块也就越多**。
- **必须手动缩小范围**，会帮助你**加速收敛问题的分析过程**，进而**快速地定位和解决问题**。


#### 作用

- 主要用于**分析绘制性能方面的问题**。
- 分析**系统关键方法和应用方法耗时**。


## 6、启动监控

### 1、实验室监控：视频录制

- 80%绘制
- 图像识别


#### 注意

覆盖高中低端机型不同的场景。


### 2、线上监控

#### 目标

需要准确地统计启动耗时。


#### 1、启动结束的统计时机

是否是使用界面显示且用户真正可以操作的时间作为启动结束时间。


#### 2、启动时间扣除逻辑

闪屏、广告和新手引导这些时间都应该从启动时间里扣除。


#### 3、启动排除逻辑

Broadcast、Server拉起，启动过程进入后台都需要排除统计。


#### 4、使用什么指标来衡量启动速度的快慢？

##### 平均启动时间的问题

一些体验很差的用户很可能被平均了。


##### 建议的指标

- 1、快开慢开比

如2s快开比，5s慢开比，可以看到**有多少比例的用户体验好，多少比例的用户比较糟糕**。


- 2、90%用户的启动时间

如果90%用户的启动时间都小于5s，那么90%区间的启动耗时就是5s。


#### 5、启动的类型有哪几种？

- 首次安装启动
- 覆盖安装启动
- 冷启动（指标）
- 热启动（反映程序的活跃或保活能力）


借鉴Facebook的 [profilo](https://github.com/facebookincubator/profilo) 工具原理，对启动整个流程进行耗时监控，在后台对不同的版本做自动化对比，监控新版本是否有新增耗时的函数。


# 四、启动优化常规方案

### 启动过程中的常见问题

- 1、**点击图标很久都不响应**：预览窗口被禁用或设置为透明。
- 2、**首页显示太慢**：初始化任务太多。
- 3、**首页显示后无法进行操作**：太多延迟初始化任务占用主线程CPU时间片。


### 优化区域

Application、Activity创建以及回调等过程。


## 1、主题切换

使用Activity的windowBackground主题属性预先设置一个启动图片（layer-list实现），在启动后，在Activity的onCreate()方法中的super.onCreate()前再setTheme(R.style.AppTheme)。


### 优点

- 使用**简单**。
- **避免了启动白屏和点击启动图标不响应的情况**。


### 缺点

- **治标不治本**，表面上产生一种快的感觉。
- **对于中低端机，总的闪屏时间会更长，建议只在Android6.0/7.0以上才启用“预览闪屏”方案，让手机性能好的用户可以有更好的体验**。


## 2、第三方库懒加载

按需初始化，特别是**针对于一些应用启动时不需要初始化的库**，可以等到用时才进行加载。


## 3、异步初始化预备知识-线程优化

### 1、Android线程调度原理剖析

#### 线程调度原理

- 1、任意时刻，**只有一个线程占用CPU，处于运行状态**。
- 2、多线程并发，轮流获取CPU使用权。
- 3、JVM负责线程调度，按照特定机制分配CPU使用权。


#### 线程调度模型

##### 1、分时调度模型

轮流获取、均分CPU。


##### 2、抢占式调度模型

优先级高的获取。


#### 如何干预线程调度？

设置线程优先级。


#### Android线程调度

##### 1、nice值

- Process中定义。
- 值越小，优先级越高。
- 默认是THREAD_PRIORITY_DEFAUT，0。


##### 2、cgroup

它是一种更严格的群组调度策略，主要分为如下两种类型：

- 后台group（默认）。
- 前台group，保证前台线程可以获取到更多的CPU


#### 注意点

- **线程过多会导致CPU频繁切换，降低线程运行效率**。
- **正确认识任务重要性以决定使用哪种线程优先级**。
- 优先级具有继承性。


### 2、Android异步方式

#### 1、Thread

- 最简单、常见的异步方式。
- 不易复用，频繁创建及销毁开销大。
- 复杂场景不易使用。


#### 2、HandlerThread

- 自带消息循环的线程。
- 串行执行。
- 长时间运行，不断从队列中获取任务。


#### 3、IntentService

- 继承自Service在内部创建HandlerThread。
- 异步，不占用主线程。
- 优先级较高，不易被系统Kill。


#### 4、AsyncTask

- Android提供的工具类。
- 无需自己处理线程切换。
- 需注意版本不一致问题（API 14以上解决）


#### 5、线程池

- Java提供的线程池。
- 易复用，减少频繁创建、销毁的时间。
- 功能强大，如定时、任务队列、并发数控制等。


#### 6、RxJava

由强大的调度器Scheduler集合提供。

不同类型的Scheduler：

- IO
- Computation


#### 异步方式总结

- 推荐度：从后往前排列。
- 正确场景选择正确的方式。


### 3、Android线程优化实战

#### 线程使用准则

- 1、**严禁使用new Thread方式**。
- 2、提供**基础线程池**供各个业务线使用，避免各个业务线各自维护一套线程池，导致线程数过多。
- 3、**根据任务类型选择合适的异步方式**：优先级低，长时间执行，HandlerThread；定时执行耗时任务，线程池。
- 4、创建线程必须**命名**，以方便**定位线程归属**，在运行期 **Thread.currentThread().setName** 修改名字。
- 5、关键异步任务监控，注意**异步不等于不耗时**，建议使用**AOP**的方式来做**监控**。
- 6、**重视优先级设置（根据任务具体情况），Process.setThreadPriority() 可以设置多次**。


### 4、如何锁定线程创建者

#### 锁定线程创建背景

- 项目变大之后收敛线程。
- 项目源码、三方库、aar中都有线程的创建。


#### 锁定线程创建方案

特别适合Hook手段，**找Hook点：构造函数或者特定方法，如Thread的构造函数。**


#### 实战

这里我们直接使用维数的 [epic](https://github.com/tiann/epic) 对Thread进行Hook。在attachBaseContext中调用DexposedBridge.hookAllConstructors方法即可，如下所示：


    DexposedBridge.hookAllConstructors(Thread.class, new XC_MethodHook() { 
        @Override protected void afterHookedMethod（MethodHookParam param）throws Throwable {                         
            super.afterHookedMethod(param); 
            Thread thread = (Thread) param.thisObject; 
            LogUtils.i("stack " + Log.getStackTraceString(new Throwable());
        }
    );
    
    
从log找到线程创建信息，根据堆栈信息跟相关业务方沟通解决方案。


### 5、线程收敛优雅实践初步

#### 线程收敛常规方案

- 根据线程创建堆栈考量合理性，使用同一线程库。
- 各业务线下掉自己的线程库。


#### 问题：基础库怎么使用线程？

直接依赖线程库，但问题在于**线程库更新可能会导致基础库更新**。


#### 基础库优雅使用线程

- **基础库内部暴露API：setExecutor**。
- **初始化的时候注入统一的线程库**。


#### 统一线程库时区分任务类型

- **IO密集型任务**：IO密集型任务不消耗CPU，核心池可以很大。常见的IO密集型任务如文件读取、写入，网络请求等等。
- **CPU密集型任务**：核心池大小和CPU核心数相关。常见的CPU密集型任务如比较复杂的计算操作，此时需要使用大量的CPU计算单元。


#### 实现用于执行多类型任务的基础线程池组件

目前基础线程池组件位于启动器sdk之中，使用非常简单，示例代码如下所示：

    // 如果当前执行的任务是CPU密集型任务，则从基础线程池组件
    // DispatcherExecutor中获取到用于执行 CPU 密集型任务的线程池
    DispatcherExecutor.getCPUExecutor().execute(YourRunable());

    // 如果当前执行的任务是IO密集型任务，则从基础线程池组件
    // DispatcherExecutor中获取到用于执行 IO 密集型任务的线程池
    DispatcherExecutor.getIOExecutor().execute(YourRunable());


具体的实现源码也比较简单，并且我对每一处代码都进行了详细的解释，就不一一具体分析了。代码如下所示：


    public class DispatcherExecutor {

        /**
         * CPU 密集型任务的线程池
         */
        private static ThreadPoolExecutor sCPUThreadPoolExecutor;
    
        /**
         * IO 密集型任务的线程池
         */
        private static ExecutorService sIOThreadPoolExecutor;
    
        /**
         * 当前设备可以使用的 CPU 核数
         */
        private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    
        /**
         * 线程池核心线程数，其数量在2 ~ 5这个区域内
         */
        private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 5));
    
        /**
         * 线程池线程数的最大值：这里指定为了核心线程数的大小
         */
        private static final int MAXIMUM_POOL_SIZE = CORE_POOL_SIZE;
    
        /**
        * 线程池中空闲线程等待工作的超时时间，当线程池中
        * 线程数量大于corePoolSize（核心线程数量）或
        * 设置了allowCoreThreadTimeOut（是否允许空闲核心线程超时）时，
        * 线程会根据keepAliveTime的值进行活性检查，一旦超时便销毁线程。
        * 否则，线程会永远等待新的工作。
        */
        private static final int KEEP_ALIVE_SECONDS = 5;
    
        /**
        * 创建一个基于链表节点的阻塞队列
        */
        private static final BlockingQueue<Runnable> S_POOL_WORK_QUEUE = new LinkedBlockingQueue<>();
    
        /**
         * 用于创建线程的线程工厂
         */
        private static final DefaultThreadFactory S_THREAD_FACTORY = new DefaultThreadFactory();
    
        /**
         * 线程池执行耗时任务时发生异常所需要做的拒绝执行处理
         * 注意：一般不会执行到这里
         */
        private static final RejectedExecutionHandler S_HANDLER = new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                Executors.newCachedThreadPool().execute(r);
            }
        };
    
        /**
         * 获取CPU线程池
         *
         * @return CPU线程池
         */
        public static ThreadPoolExecutor getCPUExecutor() {
            return sCPUThreadPoolExecutor;
        }
    
        /**
         * 获取IO线程池
         *
         * @return IO线程池
         */
        public static ExecutorService getIOExecutor() {
            return sIOThreadPoolExecutor;
        }
    
        /**
         * 实现一个默认的线程工厂
         */
        private static class DefaultThreadFactory implements ThreadFactory {
            private static final AtomicInteger POOL_NUMBER = new AtomicInteger(1);
            private final ThreadGroup group;
            private final AtomicInteger threadNumber = new AtomicInteger(1);
            private final String namePrefix;
    
            DefaultThreadFactory() {
                SecurityManager s = System.getSecurityManager();
                group = (s != null) ? s.getThreadGroup() :
                        Thread.currentThread().getThreadGroup();
                namePrefix = "TaskDispatcherPool-" +
                        POOL_NUMBER.getAndIncrement() +
                        "-Thread-";
            }
    
            @Override
            public Thread newThread(Runnable r) {
                // 每一个新创建的线程都会分配到线程组group当中
                Thread t = new Thread(group, r,
                        namePrefix + threadNumber.getAndIncrement(),
                        0);
                if (t.isDaemon()) {
                    // 非守护线程
                    t.setDaemon(false);
                }
                // 设置线程优先级
                if (t.getPriority() != Thread.NORM_PRIORITY) {
                    t.setPriority(Thread.NORM_PRIORITY);
                }
                return t;
            }
        }
    
        static {
            sCPUThreadPoolExecutor = new ThreadPoolExecutor(
                    CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                    S_POOL_WORK_QUEUE, S_THREAD_FACTORY, S_HANDLER);
            // 设置是否允许空闲核心线程超时时，线程会根据keepAliveTime的值进行活性检查，一旦超时便销毁线程。否则，线程会永远等待新的工作。
            sCPUThreadPoolExecutor.allowCoreThreadTimeOut(true);
            // IO密集型任务线程池直接采用CachedThreadPool来实现，
            // 它最多可以分配Integer.MAX_VALUE个非核心线程用来执行任务
            sIOThreadPoolExecutor = Executors.newCachedThreadPool(S_THREAD_FACTORY);
        }
    
    }


### 6、线程优化核心问题

#### 1、线程使用为什么会遇到问题？

项目发展阶段忽视基础设施建设，没有采用统一的线程池，导致线程数量过多。


##### 表现形式

异步任务执行太耗时，导致主线程卡顿。


##### 问题原因

- **1、Java线程调度是抢占式的，线程优先级比较重要，需要区分**。
- **2、没有区分IO和CPU密集型任务，导致主线程抢不到CPU**。


#### 2、怎么在项目中对线程进行优化？

##### 核心：线程收敛

- **通过Hook方式找到对应线程的堆栈信息，和业务方讨论是否应该单独起一个线程，尽可能使用统一线程池**。
- **每个基础库都暴露一个设置线程池的方法，以避免线程库更新导致基础库需要更新的问题**。
- **统一线程池应注意IO、CPU密集型任务区分**。
- 其它细节：**重要异步任务统计耗时、注重异步任务优先级和线程名的设置**。


## 4、异步初始化

### 1、核心思想

子线程分担主线程任务，并行减少时间。


### 2、异步优化注意点

- 1、**不符合异步要求**。
- 2、**需要在某个阶段完成（采用CountDownLatch确保异步任务完成后才到下一个阶段）**。
- 3、**如出现主线程要使用时还没初始化则在此次使用前初始化**。
- 4、**区分CPU密集型和IO密集型任务**。


### 3、异步初始化方案演进

- 1、new Thread
- 2、IntentService
- 3、线程池（合理配置并选择CPU密集型和IO密集型线程池）
- 4、异步启动器


### 4、异步优化最优解：异步启动器

[异步启动器源码及使用demo地址](https://github.com/zeshaoaaa/LaunchStarter)


#### 常规异步优化痛点

- 1、代码不优雅：例如使用线程池实现多个并行异步任务时会有多个executorService.submit代码块。
- 2、场景不好处理：各个初始化任务之间存在依赖关系，例如推送sdk的初始化任务需要依赖于获取设备id的初始化任务。此外，有些任务是需要在某些特定的时候就初始化完成，例如需要在Application的onCreate方法执行完之前就初始化完成。
- 3、维护成本高。


#### 启动器核心思想

**充分**利用CPU多核，**自动**梳理任务顺序。


#### 启动器流程

启动器的流程图如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99f1b02f4b9c471f9d0bfc7315b5f741~tplv-k3u1fbpfcp-zoom-1.image)


启动器的主题流程为上图中的中间区域，即**主线程与并发两个区域块**。需要注意的是，在上图中的 **head task与tail task** 并不包含在启动器的主题流程中，它仅仅是用于**处理启动前/启动后的一些通用任务**，例如我们可以在head task中做一些获取通用信息的操作，在tail task可以做一些log输出、数据上报等操作。

那么，这里我们总结一下启动的核心流程，如下所示：

- 1、**任务Task化，启动逻辑抽象成Task**（Task即对应一个个的初始化任务）。
- 2、**根据所有任务依赖关系排序生成一个有向无环图**：例如上述说到的推送SDK初始化任务需要依赖于获取设备id的初始化任务，各个任务之间都可能存在依赖关系，所以将它们的依赖关系排序生成一个有向无环图能将**并行效率最大化**。
- 3、**多线程按照排序后的优先级依次执行**：例如必须先初始化获取设备id的初始化任务，才能去进行推送SDK的初始化任务。


#### 异步启动器优化实战与源码剖析

下面，我们就来使用异步启动器来在Application的onCreate方法中进行异步优化，代码如下所示：

    
    // 1、启动器初始化
    TaskDispatcher.init(this);
    // 2、创建启动器实例，这里每次获取的都是新对象
    TaskDispatcher dispatcher = TaskDispatcher.createInstance();
    // 3、给启动器配置一系列的（异步/非异步）初始化任务并启动启动器
    dispatcher
            .addTask(new InitAMapTask())
            .addTask(new InitStethoTask())
            .addTask(new InitWeexTask())
            .addTask(new InitBuglyTask())
            .addTask(new InitFrescoTask())
            .addTask(new InitJPushTask())
            .addTask(new InitUmengTask())
            .addTask(new GetDeviceIdTask())
            .start();
            
    // 4、需要等待微信SDK初始化完成，程序才能往下执行
    dispatcher.await();


这里的 **TaskDispatcher** 就是我们的**启动器调用类**。首先，在注释1处，我们需要先调用TaskDispatcher的init方法进行启动器的初始化，其源码如下所示：


    public static void init(Context context) {
        if (context != null) {
            sContext = context;
            sHasInit = true;
            sIsMainProcess = Utils.isMainProcess(sContext);
        }
    }
    

可以看到，仅仅是初始化了几个基础字段。接着，在注释2处，我们创建了启动器实例，其源码如下所示：


    /**
     * 注意：这里我们每次获取的都是新对象
     */
    public static TaskDispatcher createInstance() {
        if (!sHasInit) {
            throw new RuntimeException("must call TaskDispatcher.init    first");
        }
        return new TaskDispatcher();
    }


在createInstance方法的中我们每次都会创建一个新的TaskDispatcher实例。然后，在注释3处，我们给启动器配置了一系列的初始化任务并启动启动器，需要注意的是，**这里的Task既可以是用于执行异步任务（子线程）的也可以是用于执行非异步任务（主线程）**。下面，我们来分析下这两种Task的用法，比如InitStethoTask这个异步任务的初始化，代码如下所示：

    
    /**
     * 异步的Task
    */
    public class InitStethoTask extends Task {

        @Override
        public void run() {
            Stetho.initializeWithDefaults(mContext);
        }
    }


这里的InitStethoTask直接继承自Task，Task中的runOnMainThread方法返回为false，说明 **task** 是用于**处理异步任务的task**，其中的run方法就是Runnable的run方法。下面，我们再看看另一个用于初始化非异步任务的例子，例如用于微信SDK初始化的InitWeexTask，代码如下所示：


    /**
    * 主线程执行的task
    */
    public class InitWeexTask extends MainTask {

        @Override
        public boolean needWait() {
            return true;
        }

        @Override
        public void run() {
            InitConfig config = new InitConfig.Builder().build();
            WXSDKEngine.initialize((Application) mContext, config);
        }
    }
    
    
可以看到，它直接继承了MainTask，MainTask的源码如下所示：

    
    public abstract class MainTask extends Task {

        @Override
        public boolean runOnMainThread() {
            return true;
        }
        
    }
    
**MainTask** 直接继承了Task，并仅仅是重写了runOnMainThread方法返回了true，说明它就是用来**初始化主线程中的非异步任务的**。

此外，我们注意到InitWeexTask中还重写了一个needWait方法并返回了true，**其目的是为了在某个时刻之前必须等待InitWeexTask初始化完成程序才能继续往下执行**，这里的某个时刻指的就是我们在Application的onCreate方法中的注释4处的代码所执行的地方：dispatcher.await()，其实现源码如下所示：


    /**
     * 需要等待的任务数
     */
    private AtomicInteger mNeedWaitCount = new AtomicInteger();

    /**
     * 调用了 await 还没结束且需要等待的任务列表
     */
    private List<Task> mNeedWaitTasks = new ArrayList<>();
    
    private CountDownLatch mCountDownLatch;
    
    private static final int WAITTIME = 10000;

    @UiThread
    public void await() {
        try {
            // 1、仅仅在测试阶段才输出需等待的任务列表数与任务名称
            if (DispatcherLog.isDebug()) {
                DispatcherLog.i("still has " + mNeedWaitCount.get());
                for (Task task : mNeedWaitTasks) {
                    DispatcherLog.i("needWait: " + task.getClass().getSimpleName());
                }
            }

            // 2、只要还有需要等待的任务没有执行完成，就调用mCountDownLatch的await方法进行等待，这里我们设定超时时间为10s
            if (mNeedWaitCount.get() > 0) {
                if (mCountDownLatch == null) {
                    throw new RuntimeException("You have to call start() before call await()");
                }
                mCountDownLatch.await(WAITTIME, TimeUnit.MILLISECONDS);
            }
        } catch (InterruptedException e) {
        }
    }

    
首先，在注释1处，我们仅仅只会在测试阶段才会输出需等待的任务列表数与任务名称。然后，在注释2处，只要需要等待的任务数mNeedWaitCount大于0，即只要还有需要等待的任务没有执行完成，就调用mCountDownLatch的await方法进行等待，注意我们这里设定了超时时间为10s。当一个**task执行完成后**，无论它是异步还是非异步的，最终都会执行到**mTaskDispatcher的markTaskDone(mTask)方法**，我们看看它的实现源码，如下所示：


    /**
    * 已经结束的Task
    */
    private volatile List<Class<? extends Task>> mFinishedTasks = new ArrayList<>(100);
    
    public void markTaskDone(Task task) {
        if (ifNeedWait(task)) {
            mFinishedTasks.add(task.getClass());
            mNeedWaitTasks.remove(task);
            mCountDownLatch.countDown();
            mNeedWaitCount.getAndDecrement();
        }
    }
    
    
可以看到，这里每执行完成一个task，就会将mCountDownLatch的锁计数减1，与此同时，也会将我们的mNeedWaitCount这个原子整数包装类的数量减1。

此外，我们在前面说到了启动器将各个任务之间的依赖关系抽象成了一个有向无环图，在上面一系列的初始化代码中，InitJPushTask是需要依赖于GetDeviceIdTask的，**那么，我们怎么告诉启动器它们两者之间的依赖关系呢**？

这里只需要在InitJPushTask中**重写dependsOn()方法，并返回包含GetDeviceIdTask的task列表即可**，代码如下所示：


    /**
    * InitJPushTask 需要在 getDeviceId 之后执行
    */
    public class InitJPushTask extends Task {

        @Override
        public List<Class<? extends Task>> dependsOn() {
            List<Class<? extends Task>> task = new ArrayList<>();
            task.add(GetDeviceIdTask.class);
            return task;
        }
    
        @Override
        public void run() {
            JPushInterface.init(mContext);
            MyApplication app = (MyApplication) mContext;
            JPushInterface.setAlias(mContext, 0, app.getDeviceId());
        }
    }
    

至此，我们的异步启动器就分析完毕了。下面我们来看看如何高效地进行延迟初始化。


## 5、延迟初始化

### 1、常规方案：利用闪屏页的停留时间进行部分初始化

- new Handler().postDelayed()。
- 界面UI展示后调用。


### 2、常规初始化痛点

- **时机不容易控制**：handler postDelayed指定的延迟时间不好估计。
- **导致界面UI卡顿**：此时用户可能还在滑动列表。


### 3、延迟优化最优解：延迟启动器

[延迟启动器源码及使用demo地址](https://github.com/zeshaoaaa/LaunchStarter)


#### 核心思想

利用IdleHandler特性，**在CPU空闲时执行，对延迟任务进行分批初始化**。


#### 延迟启动器优化实战与源码剖析

延迟初始化启动器的代码很简单，如下所示：

    /**
     * 延迟初始化分发器
     */
    public class DelayInitDispatcher {
    
        private Queue<Task> mDelayTasks = new LinkedList<>();
    
        private MessageQueue.IdleHandler mIdleHandler = new     MessageQueue.IdleHandler() {
            @Override
            public boolean queueIdle() {
                // 分批执行的好处在于每一个task占用主线程的时间相对
                // 来说很短暂，并且此时CPU是空闲的，这些能更有效地避免UI卡顿
                if(mDelayTasks.size()>0){
                    Task task = mDelayTasks.poll();
                    new DispatchRunnable(task).run();
                }
                return !mDelayTasks.isEmpty();
            }
        };
    
        public DelayInitDispatcher addTask(Task task){
            mDelayTasks.add(task);
            return this;
        }
    
        public void start(){
            Looper.myQueue().addIdleHandler(mIdleHandler);
        }

    }
    

在DelayInitDispatcher中，我们提供了mDelayTasks队列用于将每一个task添加进来，使用者只需调用addTask方法即可。**当CPU空闲时，mIdleHandler便会回调自身的queueIdle方法，这个时候我们可以将task一个一个地拿出来并执行**。这种分批执行的好处在于每一个task占用主线程的时间相对来说很短暂，并且此时CPU是空闲的，这样能更有效地避免UI卡顿，真正地提升用户的体验。

至于使用就非常简单了，我们可以直接利用SplashActivity的广告页停留时间去进行延迟初始化，代码如下所示：

    
    @Override
    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        GlobalHandler.getInstance().getHandler().post((Runnable) () -> {
            if (hasFocus) {
                DelayInitDispatcher delayInitDispatcher = new DelayInitDispatcher();
                delayInitDispatcher.addTask(new InitOtherTask())
                        .start();
            }
        });
    }
    
需要注意的是，**能异步的task我们会优先使用异步启动器在Application的onCreate方法中加载（或者是必须在Application的onCreate方法完成前必须执行完的非异task务），对于不能异步的task，我们可以利用延迟启动器进行加载。如果任务可以到用时再加载，可以使用懒加载的方式**。


#### 延迟启动器优势

- 执行时机明确。
- 缓解界面UI卡顿。
- 真正提升用户体验。


## 6、Multidex预加载优化

我们都知道，安装或者升级后首次 MultiDex 花费的时间过于漫长，我们需要进行Multidex的预加载优化。


### 1、优化步骤

- 1、启动时单独开一个进程去异步进行Multidex的第一次加载，即Dex提取和Dexopt操作。
- 2、此时，主进程Application进入while循环，不断检测Multidex操作是否完成。
- 3、执行到Multidex时，则已经发现提取并优化好了Dex，直接执行。MultiDex执行完之后主进程Application继续执行ContentProvider初始化和Application的onCreate方法。

[Multidex优化Demo地址](https://github.com/lanshifu/MultiDexTest)

#### 注意

5.0以上默认使用ART，在安装时已将Class.dex转换为oat文件了，无需优化，所以应判断只有在主进程及SDK 5.0以下才进行Multidex的预加载。


### 2、dex-opt过程是怎样的？

主要包括inline以及quick指令的优化。


#### 那么，inline是什么？

使编译器在函数调用处用函数体代码代替函数调用指令。


#### inline的作用？

函数调用的转移操作有一定的时间和空间方面的开销，特别是对于一些函数体不大且频繁调用的函数，解决其效率问题更为重要，引入inline函数就是为了解决这一问题。


#### inline又是如何进行优化的？

inline函数至少在三个方面提升了程序的时间性能：

- 1、避免了函数调用必须执行的压栈出栈等操作。
- 2、由于函数体代码被移到函数调用处，编译器可以获得更多的上下文信息，并根据这些信息对函数体代码和被调用者代码进行更进一步的优化。
- 3、若不使用inline函数，程序执行至函数调用处，需要转而去执行函数体所在位置的代码。一般函数调用位置和函数代码所在位置在代码段中并不相近，这样很容易形成操作系统的缺页中断。操作系统需要把缺页地址的代码从硬盘移入内存，所需时间将成数量级增加。而使用inline函数则可以减少缺页中断发生的机会。


#### 对于inline的使用，我们应该注意的问题？

- 1、由于inline函数在函数调用处插入函数体代码代替函数调用，若该函数在程序的很多位置被调用，有可能造成内存空间的浪费。
- 2、一般程序的压栈出栈操作也需要一定的代码，这段代码完成栈指针调整、参数传递、现场保护和恢复等操作。
若函数的函数体代码量小于编译器生成的函数压栈出栈代码，则可以放心地定义为inline，这个时候占用内存空间反而会减小。而当函数体代码大于函数压栈出栈代码时，将函数定义为inline就会增加内存空间的使用。
- 3、C++程序应该根据应用的具体场景、函数体大小、调用位置多少、函数调用的频率、应用场景对时间性能的要求，应用场景对内存性能的要求等各方面因素合理决定是否定义inline函数。
- 4、inline函数内不允许用循环语句和开关语句。


### 3、抖音BoostMultiDex优化

为了彻底解决MutiDex加载时间慢的问题，抖音团队深入挖掘了 Dalvik 虚拟机的底层系统机制，对 DEX 相关的处理逻辑进行了重新设计与优化，并推出了 BoostMultiDex 方案，它能够减少 80% 以上的黑屏等待时间，挽救低版本 Android 用户的升级安装体验。如有兴趣的同学可以看看这篇文章：[抖音BoostMultiDex优化实践：Android低版本上APP首次启动时间减少80%](https://juejin.im/post/6844904079206907911)


### 4、预加载SharedPreferences

可以利用MultiDex预加载期间的这段CPU去预加载SharedPreferences。


#### 注意

需重写getApplicationContext返回this，否则此时可能获取不到context。


## 7、类预加载优化

在Application中提前异步加载初始化耗时较长的类。


### 如何找到耗时较长的类？

替换系统的ClassLoader，打印类加载的时间，按需选取需要异步加载的类。


#### 注意

- Class.forName()只加载类本身及其静态变量的引用类。
- new 类实例 可以额外加载类成员变量的引用类。


## 8、WebView启动优化

- 1、WebView首次创建比较耗时，需要预先创建WebView提前将其内核初始化。
- 2、使用WebView缓存池，用到WebView的时候都从缓存池中拿，注意内存泄漏问题。
- 3、本地离线包，即预置静态页面资源。


## 9、页面数据预加载

在主页空闲时，将其它页面的数据加载好保存到内存或数据库，等到打开该页面时，判断已经预加载过，就直接从内存或数据库取数据并显示。


## 10、启动阶段不启动子进程

子进程会共享CPU资源，导致主进程CPU紧张。此外，在多进程情况下一定要可以在onCreate中去区分进程做一些初始化工作。


### 注意启动顺序

App onCreate之前是ContentProvider初始化。


## 11、闪屏页与主页的绘制优化

- 1、布局优化。
- 2、过渡绘制优化。


关于布局与绘制优化可以参考[Android性能优化之绘制优化](https://juejin.im/post/6844904080989487118)。


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78bbd572ce5e4318a5162ae8f9a588c1~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、[Android开发高手课之启动优化](https://time.geekbang.org/column/article/73651)

2、[支付宝客户端架构解析：Android
客户端启动速度优化之「垃圾回收」](https://yq.aliyun.com/articles/672750)

3、[支付宝 App 构建优化解析：通过安装包重排布优化 Android 端启动性能](https://yq.aliyun.com/articles/680526)

4、[Facebook Redex字节码优化工具](https://github.com/facebook/redex)

5、[微信Android热补丁实践演进之路](https://blog.csdn.net/tencent_bugly/article/details/51821722)

6、[安卓App热补丁动态修复技术介绍](https://zhuanlan.zhihu.com/p/20308548)

7、[Dalvik Optimization and Verification With dexopt](https://blog.csdn.net/fishmai/article/details/52398485)

8、[微信在Github开源了Hardcoder，对Android开发者有什么影响？](https://github.com/Tencent/Hardcoder)

9、[历时三年研发，OPPO 的 Hyper Boost 引擎如何对系统、游戏和应用实现加速？](http://mobile.yesky.com/433/311567433.shtml)

10、[抱歉，Xposed真的可以为所欲为](https://blog.csdn.net/coder_pig/article/details/80031285)

11、[墙上时钟时间 ，用户cpu时间 ，系统cpu时间的理解](https://blog.csdn.net/balian8/article/details/53319580)  

12、《Android应用性能优化最佳实践》

13、[必知必会 | Android 性能优化的方面方面都在这儿](https://mp.weixin.qq.com/s/QVOYF2nfoWMCbM5YsxQgRQ?)

14、[极客时间之Top团队大牛带你玩转Android性能分析与优化](https://coding.imooc.com/class/308.html)

15、[启动器源码](https://github.com/zeshaoaaa/LaunchStarter)

16、[MultiDex优化源码](https://github.com/lanshifu/MultiDexTest)

17、[使用gradle自动化增加Trace Tag](https://github.com/AndroidAdvanceWithGeektime/Chapter07)


## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f179e593fc004ae9bce39875b29d57e3~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>


###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。
