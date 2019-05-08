


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