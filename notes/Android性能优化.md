


# App启动速度优化

## 一、认识启动加速

### 含义

从点击图标到用户可操作的全部过程

### 意义

避免用户一安装应用就卸载

### 分类

- 冷启动
- 热启动
- 温启动

### 过程

#### 冷启动前

- 1、点击相应应用解析
- 2、App启动之后立即展示一个空白的Window（预览窗口显示）
- 3、创建App进程

#### 冷启动后

- 1、创建App对象
- 2、启动Main Thread
- 3、创建启动的Activity对象，闪屏显示
- 4、创建启动的MainActivity对象，主页显示
- 5、其它工作

## 二、优化工具

力求获取准确的数据评估

### 1、TraceView

性能损耗太大，得出的结果并不真实

#### 作用：

主要做热点分析，得到两种数据

- 单次执行最耗时的方法
- 执行次数最多的方法

#### 使用：

- 1、代码中添加：Debug.startMethodTracing()、检测方法、Debug.stopMethodTracing()
- 2、打开Profile->CPU->点击Record->点击Stop->查看Profile下方Top Down/Bottom Up找出耗时的热点方法。

### 2、Systrace+函数插桩

#### Systrace原理

在系统的一些关键链路（如SystemServcie、虚拟机、Binder驱动）插入一些信息（Label），
通过Label的开始和结束来确定某个核心过程的执行时间，然后把这些Label信息收集起来得到系统关键路径的运行时间信息，
最后得到整个系统的运行性能信息。Android Framework里面一些重要的模块都插入了label信息(Java层通过android.os.Trace类完成，
native层通过ATrace宏完成），用户App中可以添加自定义的Lable。

#### 特性

- 系统版本越高，Android Framework中添加的系统可用Label就越多，能够支持和分析的系统模块也就越多。
- 必须手动缩小范围，会帮助你加速收敛问题的分析过程，进而快速地定位和解决问题。

#### 使用方式：

##### 1、命令行

systrace.py -t 10 sched gfx view wm am app webview -a <package-name>

##### 2、代码插桩

定义Trace静态工厂类i，o封装Trace.begainSection(),Trace.endSection()。

分析App的冷启动过程：在Application类的attachBaseContext调用`Trace.beginSection("Boot Procedure")`，然后在App首页的onWindowFoucusChanged方法
或者你认为别的合适的启动结束点调用`Trace.endSection`就可以到启动过程的信息。

手动开启App自定义Label的Trace功能：通过反射设置setAppTracingAllowed方法为true。

## 三、启动加速优化

### 常见问题

#### 1、点击图标很久都不响应

预览窗口被禁用或设置为透明

#### 2、首页显示太慢

初始化任务太多

#### 3、首页显示后无法操作

太多任务初始化异步/延迟

### 优化区域

Application、Activity创建以及回调等过程

### 通常手段

- 利用主题背景防止出现白屏
- 减少Application的onCreate中所要做的事情
- 一些不重要的SDK延迟或者异步加载
- 多进程情况下一定要可以在onCreate中去区分进程做一些初始化工作
- 部分将要使用到的类异步加载（IntentService）
- 针对multidex专门做优化（5.0以上默认使用ART，在安装时已将Class.dex转换为oat文件了，无需优化）

### 优化步骤

#### 一、主题切换

使用Activity的windowBackground主题属性预先设置一个启动图片（layer-list）,在启动后，在Activity的onCreate()方法前再setTheme(R.style.AppTheme)

优点：简单

缺点：
治标不治本，表面上产生一种快的感觉。

对于中低端机，总的闪屏时间会更长，建议只在Android6.0/7.0以上才启用“预览闪屏”方案，让手机性能好的用户可以有更好的体验。

#### 二、避免过重的App初始化

获取所有应用activity堆栈信息

adb shell dumpsys activity

adb获取自己的应用activity堆栈信息

adb shell dumpsys activity | grep com.xxx.xxx.xx

使用adb命令启动一个Activity

adb shell am start com.growingwiththeweb.example/.MainActivity

adb命令统计启动时间

adb shell am start -W com.growingwiththeweb.example/.SplashActivity

- 1、分步初始化（优先级高的放前面）
- 2、异步初始化（如出现主线程要使用时还没初始化则在此次使用前初始化）
- 3、延迟初始化（利用闪屏页的停留时间进行部分初始化）

#### 三、Multidex预加载优化

- 1、启动时单独开一个进程进行Multidex的第一次加载，即Dex提取和Dexopt操作。
- 2、此时，主进程在后台等待，优化进程执行完毕后通知主进程继续执行，此时执行到Multidex时，则已经发现提取优化好了Dex，直接执行。

注意：判断只有在主进程及SDK 5.0以下才进行Multidex的预加载。

#### 四、闪屏页和主页绘制优化

#### 五、业务层面优化


# App内存优化

## 一、基础：

### 1、shell命令分析内存状况（不同手机的App进程分配的最大内存不一样）：

dvm最大可用内存：adb shell getprop | grep dalvik.vm.heapsize

单个程序限制最大可用内存(如果设置了开启largeHeap，则可提高到dvm最大内存才OOM）：adb shell getprop | grep heapgrowthlimit

输出App内存使用情况概览：adb shell dumpsys meminfo 包名 ，重要字段含义如下：

- Pss：此进程内存+其他进程共享内存（按比例分配）
- Private Dirty：此进程独享内存
- Heap Size：分配的内存
- Heap Alloc：已使用的内存
- Heap Free：空闲内存

### 2、查看Profile，可知优化的点为：

- JavaHeap
- NativeHeap
- Stack
- Code

### 3、MAT使用

1、在使用MAT之前，先使用as的Profile中的Memory去获取要分析的堆内存快照文件.hprof，如果要测试某个页面是否产生内存泄漏，可以先dump出没进入该页面的内存快照文件.hprof，然后，通常执行5次进入/退出该页面，然后再dump出此刻的内存快照文件.hprof，最后，将两者比较，如果内存相差明显，则可能发生内存泄露。（注意:MAT需要标准的.hprof文件，因此在as的Profiler中GC后dump出的内存快照文件.hprof必须手动使用android sdk platform-tools下的hprof-conv程序进行转换才能被MAT打开）

2、然后，使用MAT打开前面保存的2份.hprof文件，打开Overview界面，在Overview界面下面有4种action，其中最常用的就是Histogram和Dominator Tree。
Dominator Tree：支配树，按对象大小降序列出对象和其所引用的对象，注重引用关系分析。选择Group by package，找到当前要检测的类（或者使用顶部的Regex直接搜索），查看它的Object数目是否正确，如果多了，则判断发生了内存泄露。然后，点击右击该类，选择Merge Shortest Paths to GC Root中的exclude all phantom/weak/soft etc.references选项来查看该类的GC强引用链。最后，通过引用链即可看到最终强引用该类的对象。

Histogram：直方图注重量的分析。使用方式与Dominator Tree类似。

3、对比hprof文件，检测出复杂情况下的内存泄露：

通用对比方式：在Navigation History下面选择想要对比的dominator_tree/histogram，右击选择Add to Compare Basket，然后在Compare Basket一栏中点击红色感叹号（Compare the results）生成对比表格（Compared Tables），在顶部Regex输入要检测的类，查看引用关系或对象数量去进行分析即可。

针对于Histogram的快速对比方式：直接选择Histogram上方的Compare to another Heap Dump选择要比较的hprof文件的Histogram即可。

## 二、内存优化：

### 内存泄露的本质

无法回收无用的对象

### 常见的内存泄漏案例：

- 1、单例造成的内存泄漏（使用Application的Context）
- 2、非静态内部类创建静态实例造成的内存泄漏（将该内部类设为静态内部类或将该内部类抽取出来封装成一个单例，如果需要使用Context，就使用Application的Context）
- 3、Handler造成的内存泄漏（将Handler类独立出来或者使用静态内部类）
- 4、线程造成的内存泄漏（将AsyncTask和Runnable类独立出来或者使用静态内部类）
- 5、BraodcastReceiver、Bitmap等资源未关闭造成的内存泄漏（应该在Activity销毁时及时关闭或者注销）
- 6、使用ListView时造成的内存泄漏（在构造Adapter时，使用缓存的convertView）
- 7、集合容器中的内存泄露（在退出程序之前，将集合里的东西clear，然后置为null，再退出程序）
- 8、WebView造成的泄露（为WebView另外开启一个进程，通过AIDL与主线程进行通信，WebView所在的进程可以根据业务的需要选择合适的时机进行销毁，从而达到内存的完整释放）

### 常见的内存优化点：

1、只需要UI提供一套高分辨率的图，图片建议放在drawable-xxhdpi文件夹下，这样在低分辨率设备中图片的大小只是压缩，不会存在内存增大的情况。如若遇到不需缩放的文件，放在drawable-nodpi文件夹下。

2、图片优化：

- 颜色模式：RGB_8888->RGB_565
- 降低图片大小
- 降低采样率

3、在App退到后台内存紧张即将被Kill掉时选择重写onTrimMemory()方法去释放掉图片缓存、静态缓存来自保。

4、item被回收不可见时释放掉对图片的引用：

- ListView：因此每次item被回收后再次利用都会重新绑定数据，只需在ImageView onDetachFromWindow的时候释放掉图片引用即可。
- RecyclerView：因为被回收不可见时第一选择是放进mCacheView中，这里item被复用并不会只需bindViewHolder来重新绑定数据，只有被回收进mRecyclePool中后拿出来复用才会重新绑定数据，因此重写Recycler.Adapter中的onViewRecycled()方法来使item被回收进RecyclePool的时候去释放图片引用。

5、集合优化：Android提供了一系列优化过后的数据集合工具类，如SparseArray、SparseBooleanArray、LongSparseArray，使用这些API可以让我们的程序更加高效。HashMap工具类会相对比较低效，因为它需要为每一个键值对都提供一个对象入口，而SparseArray就避免掉了基本数据类型转换成对象数据类型的时间。

6、避免创作不必要的对象：字符串拼接使用StringBuffer，StringBuilder。

7、onDraw方法里面不要执行对象的创建。

8、使用static final 优化成员变量。

9、使用增强型for循环语法。

10、在没有特殊原因的情况下，尽量使用基本数据类型来代替封装数据类型，int比Integer要更加有效，其它数据类型也是一样。

11、适当采用软引用和弱引用。

12、采用内存缓存和磁盘缓存。

13、尽量采用静态内部类，可避免潜在由于内部类导致的内存泄漏。

## 三、检测内存泄露方式：

1、shell命令+LeakCanary+MAT（时间充足时使用）：运行程序，所有功能跑一遍，确保没有改出问题，完全退出程序，手动触发GC，然后使用adb shell dumpsys meminfo packagename -d命令查看退出界面后Objects下的Views和Activities数目是否为0，如果不是则通过LeakCanary检查可能存在内存泄露的地方，最后通过MAT分析，如此反复，改善满意为止。

2、Profile MEMORY（时间有限时使用）：运行程序，对每一个页面进行内存分析检查。首先，反复打开关闭页面5次，然后收到GC（点击Profile MEMORY左上角的垃圾桶图标），如果此时total内存还没有恢复到之前的数值，则可能发生了内存泄露。此时，再点击Profile MEMORY左上角的垃圾桶图标旁的heap dump按钮查看当前的内存堆栈情况，选择按包名查找，找到当前测试的Activity，如果引用了多个实例，则表明发生了内存泄露。





