<!-- GFM-TOC -->
* [一、App启动速度优化](#App启动速度优化)
* [二、App内存优化](#App内存优化)
* [三、App绘制优化](#App绘制优化)
* [四、App瘦身](#App瘦身)
* [五、APP电量优化](#APP电量优化)


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

- 1、点击相应应用图标
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


# App绘制优化

## 一、认识绘制优化

确保刷新帧率>60fps,完成单次刷新<16ms。

### 目标

保证稳定的帧率来避免卡顿。

## 二、检测性能瓶颈（优化工具）

### 一、Layout Inspector

只能分析Android Studio中正在运行的App的视图布局结构。

### 二、GPU配置渲染工具

从Android M开始渲染八步骤：

#### 1、橙色-Swap Buffers

表示GPU处理任务的时间。

#### 2、红色-Command Issue

进行2D渲染显示列表的时间，越高表示需要绘制的视图越多。

#### 3、浅蓝-Sync&Upload

准备有待绘制的图片所耗费的时间，越高表示图片数量越多或图片越大。

#### 4、深蓝-Draw

测量和绘制视图所需的时间，越高表示视图越多或onDraw方法有耗时操作。

#### 5、一级绿-Measure/Layout

onMeasure与onLayout所花费的时间。

#### 6、二级绿-Animation

执行动画所需要花费的时间。越高表示使用了非官方动画工具或执行中有读写操作。

#### 7、三级绿-Input Handling

系统处理输入事件所耗费的时间。

#### 8、四级绿-Misc Time/Vsync Delay

主线程执行了太多任务，导致UI渲染跟不上vSync的信号而出现掉帧。

### 三、卡顿检测工具

#### 1、监听Looper日志实现

AndroidperformanceMonitor(BlockCanary)

#### 2、利用Choreographer

Takt

TinyDancer

## 三、绘制优化

### 一、过度绘制

GPU过度绘制5中区域颜色：

- 原色：无过度绘制。
- 蓝色：1次。
- 绿色：2次。
- 粉色：3次。
- 红色：4次。
- 标准：蓝色或以下。

#### 方式

- 1、处理有图片背景的区域在代码里有选择性的取出背景。
- 2、有选择性地移除窗口背景：getWindow().setBackgroundDrawable(null)。
- 3、自定义View中使用Canvas.clipRect。

### 二、减少嵌套层次和控件个数

#### 布局加载原理

LayoutInflater通过pull解析方式来解析各个xml节点，再将对应的各个节点名使用反射创建出View的对象实例。

#### 优化顺序

- 1、使用TextView替换RL、LL。
- 2、使用低端机进行优化，以发现性能瓶颈。
- 3、使用merge、ViewStub标签。
- 4、onDraw不能进行复杂运算。
- 5、使用TextView的行间距替换多行文本：lineSpacingExtra/lineSpacingMultiplier。
- 6、使用Spannable/Html.fromHtml替换多种不同规格文字。
- 7、用LinearLayout自带的分割线。
- 8、使用Space添加间距。
- 9、lint + alibaba规约修复点。
- 10、使用约束布局。
- 11、Common Style抽取 (自定义快捷键style：Ctrl + Alt+ commad-2）
- 12、布局属性排列顺序规范（快捷键Ctrl + Alt + F）


# App瘦身

## 一、分析APK组成结构：

1、打开build下的analyze apk选项，选择当前的release apk。

2、apk的组成为：

- dex
- res
- asserts
- lib
- META-INF：存放签名信息
- MANIFEST.MF：其中每一个资源文件都有一个SHA-256-Digest签名，MANIFEST.MF文件的SHA256（SHA1）并base64编码的结果即为CERT.SF中的SHA256-Digest-Manifest值。
- CERT.SF：除了开头处定义的SHA256（SHA1）-Digest-Manifest值，后面几项的值是对MANIFEST.MF文件中的每项再次SHA256并base64编码后的值。
- CERT.RSA：其中包含了公钥、加密算法等信息。首先对前一步生成的MANIFEST.MF使用了SHA256（SHA1）-RSA算法，用开发者私钥签名，然后在安装时使用公钥解密。最后，将其与未加密的摘要信息（MANIFEST.MF文件）进行对比，如果相符，则表明内容没有被修改。
- androidManifest
- resources.arsc：编译后的二进制资源文件

3、瘦身完毕后，可利用分析工具的Compare with进行apk优化前后的对比。

## 二、apk优化：

- 1、assets：删除其中的无用资源和无用字体。
- 2、使用svg替代icon-font。
- 3、动态下载资源：例如字体、js代码、图片。
- 4、压缩资源文件，用到的时候再动态解压。
- 5、优化lib：配置abiFilters，使用armeabi、aremabi-v7a、×86即可，mips属于小众，默认支持arm的so的，但×86的不支持。
- 6、避免复制so：Android 6.0之前，so文件会压缩到apk中。系统安装应用的时候，会把so文件解压到data分区，这会导致多占用一倍的空间。在6.0+中，使用：<application android:extractnativeLibs="false"...>。
- 7、删除无用的资源映射。
- 8、对res进行资源名称混淆：AndResGuard。
- 9、右击as的任何文件，选择Refactor->Remove Unused Resources删除无用资源（不要勾选清除id）。或者在打包时进行收缩资源，设置buildType {release { shrinkResources true} } 即可。
- 10、如果国内应用可以只支持中文，通过设置defaultConfig { resConfigs "zh" } }即可。
- 11、统一应用风格：如统一颜色和按钮按压效果。
- 12、图片放置优化思路：
  
    1、聊天表情出一套图 -> hdpi

    2、纯色小icon使用VD -> raw

    3、背景大图出一套 ->xhdpi

    4、logo等权重比较大的图片出两套 ->hdpi,xhdpi

    5、若某些图在真机中有异常，则用多套图

    6、若遇到奇葩机型，则针对性补图

- 13、优化图片：

    谷歌使用建议：VD（纯色icon）->WebP（非纯色icon）->Png（更好效果） ->jpg（若无alpha通道）
    
    使用VectorDrawable：
    注意点：
    1、必须通过app:arcCompat属性来使用svg，如果通过src，则在低版本手机上会出现不兼容的问题。
    2、可能会不兼容selector，在Activity中手动兼容即可：static { AppCompatDelegate.setCompatVectorFromResourcesEnabled(true) }
    3、不兼容第三方库
    4、性能问题：当Vector比较简单时，效率肯定比Bitmap高，复杂则效率会不如Bitmap。
    5、不便于管理：建议原则为，同目录多类型文件，以前缀区别，不同目录相同类型文件，以意义区分。
    
    使用WebP：
    注意点：
    1、兼容性不好，最低支持4.2.1。
    2、不便于预览，需使用浏览器打开。
    3、可使用WebpConvertPlugin这个gradle插件在mergeXXXResource Task和processXXXResource Task直接插入一个task去将png、jpg图片批量替换成webp图片。
    
- 14、复用按压效果：

    分割线使用统一的shape。

    item背景使用统一的selector。
    
- 15、压缩图片：

    优先压缩大图（建议有损压缩），后小图，不压.9图。
    发版前通过TinyPIC_Gradle_Plugin插件来使用tinypng压缩一次图片。注意：app默认会在打包时进行图片的压缩工作，此时建议手动禁止：android {           defaultConfig { ... } aaptOptions { cruncherEnabled = false } }
    
- 16、使用Lint，在as的Inspect Code对工程做静态代码检查，去除无用代码和优化不规范代码。
- 17、开启混淆。


# APP电量优化

## 一、电量测试

1、注册action：ACTION_BATTERY_CHANGED广播去获取手机整体耗电量信息，但实时性差，精度低。

2、使用Battery Historian：

Android5.0之后Google开源的一款检测电池有关的信息和事件的工具，可可视化分析相关指标如耗电比例、Wifi、蜂窝数据量、WakeLock唤醒次数。Android6.0更新的2.0新增了引起手机状态变化的应用。此外，它还可以筛选出特定的App查看每个模块的耗时及执行次数。

## 二、电量优化

1、使用TraceView定位CPU占用率异常的问题，对CPU时间片优化。

2、网络传输：

- 1、通过数据压缩减少传输数据，降低电量消耗
- 2、尽量选择更快的传输方式如WIFI
- 3、可以将分析和统计等非重要操作等到电量充足或Wifi状态时去请求集中发送
- 4、无网状态避免网络请求

3、根据不同场景和不同类型的App对定位进行个性化的区分：

- 从GPS_PROVIDER、NETWORK_PROVIDER、PASSIVE_PROVIDER选择合适的Location Provider或者使用Criteria，设置合适的模式、功耗、海波、速度等需求，系统会返回合适的Location Provider。
- 获取到定位或程序位于后台时，及时注销定位监听。
- 多模块使用定位尽量复用。

4、谨慎使用WakeLock，它是一种锁的机制，只要有app进程持有一个WakeLock，手机就不会休眠：

- App前台不申请WakeLock
- App后台要申请时使用带超时参数的方法，防止由于忘记或异常情况下没有释放
- 任务结束及时释放
- 屏幕常量使用FLAG_KEEP_SCREEN_ON即可

5、传感器使用：

- 从SENSOR_DELAY_NORMAL(200000微秒）、SENSOR_DELAY_UI(60000微秒）、SENSOR_DELAY_GAME(20000微秒)、SENSOR_DELAY_FASTEST(0微秒）中选用合适采样率的传感器
- 后台及时注销传感器监听

6、在Api21即以上使用Job Scheduler，使用场景如下：

- 1、应用可推迟的非面向用户的工作：定期数据库数据更新
- 2、充电时才希望执行的工作
- 3、访问网络或Wi-Fi连接时需要进行的任务
- 4、将多个任务打包在需要的场景下执行

## 三、优化步骤

1、在设置-电池里查看App的耗电情况。

2、使用Battery Historian进行分析。

3、根据分析结果选用上面的电量优化方式进行优化。

