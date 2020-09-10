---

		title:  深入探索 Android 电量优化
		date: 2020/2/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---


# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


# 本文思维导图

![](https://user-gold-cdn.xitu.io/2020/6/16/172ba7fbb74412e8?w=3210&h=1328&f=png&s=520243)   


# 一、正确认识

## 1、为什么要做电量优化？

在 Android 应用开发中，我们需要考虑的是如何优化电量使用，让我们的 App 不会因为电量消耗过高被用户排斥，或者被其他安全应用报告，以此确保用户黏性。

## 2、电量重视度不够

开发中一直连接手机，不知道电量消耗有多快。

## 3、电量消耗线上难以量化

我们没有办法拿到每一个用户手机的组件能耗，其中不同的硬件模块使用了不同的参数，然后使用了不同的算法来进行估算。但是，具体的参数值根据手机所使用的硬件来说是不一样的。


# 二、电池技术

## 1、电池容量

现在一般手机的电池容量会占用内部组件将近一半的空间。

## 2、充电时间

### 1）、OPPO VOOC 闪充技术

- 1、适配器中加入 MCU 智能芯片，得益于 MCU 对电流的精准调节，VOOC 实现了分段恒流和分档技术，起步时，VOOC 会挂上高速档，中间时会自动挂上中速档，让你快速前行，结尾时又会切换成低速挡，让你平稳到站。
- 2、从适配器到接口再到手机内部的全端式五重防护技术。
    - 1）、适配器过载保护
电流进入适配器时，其中的传感器会实时检测电压电流，安全时， MOSFET（保护）开关会自动打开闪充。
    - 2）、闪充条件鉴定保护
电流通过适配器时，MCU 芯片会识别设备是否支持闪充，只有支持才开启闪充与第二级过载保护。
    - 3）、接口过载保护
电流进入手机时，在特别定制的 7pinUSB 接口处，手机内的 MCU 会控制第三个 MOSFET（保护）开关，实行第三级过载保护。
    - 4）、电池过载保护
电池内的特殊 IC 和 MOSFET（保护）开关负责对进入电池的电压电流实行过载保护。
    - 5）、电池熔丝保护
出现异常时，电池内的保险丝会立即熔断，物理性断绝电流输入。
- 3、将充电安全指数从 PPM（百万分之一）提升至航天级别 DPM（十亿分之一）。


### 2）、快充技术

P=UI（电功率=电压 * 电流）

#### 普通充电过程

- 1）、先将 220V 电压通过充电头降至 5V。
- 2）、然后，手机内部电路再把 5V 电压降至 4.2V。
- 3）、最后，把电量输送给电池，而整个降压的过程中会产生热能。


#### 分类

- 1）、高压低电流快充方案：在充电过程中国提升充电电压（7-20V）来提升充电功率。
- 2）、低压大电流快充方案：在电压一定情况下，增加电流，通常使用并联电路的方式进行分流。


### 3）、铝-石墨烯超级电池

- 超高耐用性和安全性，快充充电1.1秒就能充满电。
- 实验阶段。


## 3、寿命

通常使用充电循环次数衡量。


## 4、安全性

严格控制电池容量，例如 VOOC 就使用了各种安全检测技术。


## 5、电量和硬件

- 手机耗电是通过使用相应的硬件模块来消耗电能。
- CPU、屏幕、WIFI、数据网络、GPS、音视频通话在日常耗电量中占比最大。


## 6、Android 耗电演进

### KITKAT

#### 批处理传感器

分批有效地收集和传递传感器事件。

#### Alarm 对齐

批处理在合理的相似时间内的所有应用的闹铃，以便系统仅唤醒一次。

### Lollipop

- 开启 Volta 项目
- Job Scheduler
- dumpsys batterystats
- Battery Historian
- 修复 native fork 进程保活的 bug


### Marshmallow

- 省电功能
- Doze 低功耗模式
- App Standby 应用待机摸手机


### Nougat

- 优化省电功能
- Doze 加强版
- implicit broadcasts 显示
- 混合编译


### Oreo

- 更多优化省电功能
- 后台执行限制
- 后台位置限制


### P（电压管理严格限制）

#### 应用待机分组（App Standby Bueckets）

- 从应用安装开始。
- 分组决定后台被限制的程度。
- 不常用的应用将被限制地更加严格。


#### 应用后台限制（Background Restrictions）

- 用户开启。
- 停止后台运行。
- 提示用户后台耗电严重的应用，用户可选择停止它们的后台运行。


#### 省电模式（Battery Saver）

- 用户开启。
- 所有应用进入待机模式。
- 更加严格的后台限制，而且无视应用的 Target API。


# 三、电量检测方案

对于电量的统计有一个公式，如下所示：

```
模块电量（mAh） = 模块电流（mA）* 模块耗时（h）
```

Android 系统要求 ROM 厂商必须在 /frameworks/base/core/res/res/xml/power_profile.xml 提供组件的电源配置文件。而 Android 系统的电量计算 PowerProfile 正是通过读取 power_profile.xml 的数据。


## 1、设置—耗电排行

- 1）、直观，但没有详细数据，对解决问题帮助不大。
- 2）、需要找特定场景专项测试，比如在某一个界面操作一段时间，然后来判断这个页面是否耗电。

## 2、使用广播监听电量变化—ACTION_BATTERY_CHANGED

获取电池电量、充电状态、电池状态等信息。

### 实战案例

```java
IntentFilter filter = new IntentFilter();
filter.addAction(Intent.ACTION_BATTERY_CHANGED);
Intent intent = registerReceiver(null, filter);
LogUtils.i("battery " + intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1));
```


### 缺点

- 1）、价值不大：针对手机整体的耗电量，而非单个 App。
- 2）、实时性差、精度较低，被动通知。


## 3、dumpsys batterystats

batterystats 是 Android 5.0 提供的工具，它可以获取各个 App 的 WakeLock、CPU 时间占用等信息，同时增加了一个 Estimated power use（mAh）功能，预估耗电量。

### 作用

将电量测量转化为功能模块的使用时间或者次数。

```java
adb shell dumpsys batterystats > battery.txt
```


在 battery.txt 搜索 ‘Estimated power use’ 关键字，下面粗略统计了各个 Uid 的总耗电量。

```java
Estimated power use (mAh):
Capacity: 3350, Computed drain: 2767, actual drain: 3752-3853
Uid 1000: 1014 ( cpu=999 wake=1.36 radio=11.4 wifi=1.24 gps=0.435 sensor=0.808 ) Excluded from smearing
Unaccounted: 985 ( ) Including smearing: 0 ( ) Excluded from smearing
Uid 0: 416 ( cpu=157 wake=210 radio=38.8 wifi=9.51 ) Excluded from smearing
...
```


batterystats 所记录的电量统计数据源自于 BatteryStatsService-电量统计服务，其实现类为 BatteryStatsImpl，内部正是使用的 PowerProfile 。

BatteryStatsImpl 为每一个应用创建与之对应的 UID 来监控器系统资源的使用情况，其统计了 12 大模块的电量消耗，如下所示：

- Camera、Audio、Video
- Bluetooth、Network、Wakelock
- Sensor、Radio、Screen
- WIFI、CPU、GPS




## 4、[Battery Historian](https://github.com/google/battery-historian)

### 特点

- 1）、查看自设备上次充电以来各种汇总统计信息，而且可以选择对应的 App 查看详细信息。
- 2）、可视化展示指标：
    - 耗电比例。
    - 执行时间、次数。
- 3）、仅适合线下使用。

### 安装

- 1）、安装 Docker
- 2）、docker -- run -p <port>:9999 gcr.io/android-battery-historian/stable:3.0 --port 9999 （需要外网）


### 导出电量信息

- 1）、使用 batterystats 命令重置手机电量：adb shell dumpsys batterystats --reset
- 2)、使用 batterystats 命令获取电池数据权限并开启记录全面的电量信息：adb shell dumpsys batterystats --enable full-wake-history
- 3）、测试完成后，使用 bugreport 导出电量信息：
    - 7.0和7.0以后：adb bugreport bugreport.zip
    - 6.0和6.0之前:adb bugreport > bugreport.txt
    - 通过 historian 图形化展示结果：python historian.py -a bugreport.txt > battery.html


### 上传分析

- 1）、打开 http://localhost:<port>

![](https://user-gold-cdn.xitu.io/2020/6/7/1728dc7d4df779e2?w=1500&h=594&f=png&s=84722)


如果打不开，可以使用备用网站 [bathist](https://bathist.ef.lc/)


- 2）、上传 bugreport 文件，点 Submit 提交即可。


### Battery Historian 数据分析

#### Hitorian V2 — 电量统计图表

##### Add Metrics

![](https://user-gold-cdn.xitu.io/2020/6/7/1728dc804811b00e?w=572&h=634&f=png&s=79148)


在 Add Metrics 中我们可以增加更多的测量项。

##### CPU running

![](https://user-gold-cdn.xitu.io/2020/6/7/1728dc8324616778?w=2866&h=1488&f=png&s=448392)


如果一直处于 running，则表明电量消耗比较高。

##### JobScheduler

![](https://user-gold-cdn.xitu.io/2020/6/7/1728dc86896f8829?w=2876&h=1452&f=png&s=438761)


选中 Job Scheduler 的某一个工作时间片，我们可以查看具体的 发生的时间、耗时以及次数，最重要的是它统计出来了是哪一个进程在使用这个 JobScheduler。

#### App Selection

![](https://user-gold-cdn.xitu.io/2020/6/7/1728dc892e1b6384?w=898&h=490&f=png&s=78941)


- 1）、选择要分析电量的指定 App。
- 2）、点击右边区域的 System Stats 一栏可以在下方查看各个系统组件的电量百分比消耗详情，例如 Userspace Wakelocks。


#### 主入口处的 Switch to Bugreport Comparison

![](https://user-gold-cdn.xitu.io/2020/6/7/1728dc8bc221bdab?w=1438&h=478&f=png&s=73877)


选择多个文件进行上传对比。


## 5、电量专项测试

### 1）、耗电场景测试

- 复杂计算。
- 音视频播放。


### 2）、传感器相关

- 使用时长
- 耗电量
- 发热


### 3）、后台静默测试


# 四、耗电优化

## 1、耗电优化的难点

- 1）、**缺乏现场，无法复现**。
- 2）、**信息不全，难以定位**。
- 3）、**无法评估结果**。


在 App 开发中，经常会由于某个需求场景或 代码 bug 而导致大量耗电。

## 2、后台调度任务省电

### 思考步骤

- 需要后台运行
    - 长时间下载：DownloadManager
    - 数据同步：SyncAdapter
    - 本地任务：JobScheduler
- 特定时间执行：AlarmManager
- 实时通信：推送服务
- 立刻执行：Foreground Service


对于耗电优化中，我们最常用的就是 JobScheduler，下面👇，我们来实战一下。

### Job Scheduler 实战

```java
/**
 * 开启 JobScheduler
 */
private void startJobScheduler() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        JobScheduler jobScheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
        JobInfo.Builder builder = new JobInfo.Builder(1, new ComponentName(getPackageName(), JobSchedulerService.class.getName()));
        // 设置仅在 充电和WIFI 下才使用 JobScheduler 进行批量任务处理
        builder.setRequiresCharging(true)
                .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED);
        jobScheduler.schedule(builder.build());
    }
}
```


其中，**JobSchedulerService 就是用于进行批量任务处理的服务**，示例代码如下所示：


```java
/**
 * 用于进行批量任务处理的 JobSchedulerService
 */
@RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
public class JobSchedulerService extends JobService {

    @Override
    public boolean onStartJob(JobParameters params) {
        // 此处执行在主线程
        // 模拟一些处理：批量网络请求，APM日志上报
        return false;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        return false;
    }
}
```


#### 特点

- 1）、**仅支持 API 21 及之上**。
- 2）、**在符合某些条件时创建执行在后台的任务**。
- 3）、**把不紧急的任务放到更合适的时机批量处理**。


符合 Android 规则，手机在充电状态才去做耗电工作。示例代码如下所示：

```java
IntentFilter ifilter = new IntentFilter(Intent.ACTION_BATTERY_CHANGED);
Intent batteryStatus = context.registerReceiver(null, ifilter);
//获取用户是否在充电的状态或者已经充满电了
int status = batteryStatus.getIntExtra(BatteryManager.EXTRA_STATUS, -1);
boolean isCharging = status == BatteryManager.BATTERY_STATUS_CHARGING || status == BatteryManager.BATTERY_STATUS_FULL;
```


## 3、电量优化套路总结

### 1、优化应用的后台耗电

避免后台长时间获取 WakeLock、WIFI 和蓝牙的扫描等。

### 2、符合系统的耗电规则

Android P 使用了 Android Vitals 监控后台耗电，其规则如下所示：

- 1）、Alarm Manager wakeup 唤醒过多：当手机不在充电状态，每小时 wakeup 唤醒次数大于 10 次。
- 2）、频繁使用局部唤醒锁：当手机不在充电状态，partial wake lock 持有超过1小时。
- 3）、后台网络使用量过高：当手机不在充电状态而且应用在后台，每小时网络使用量超过 50MB。
- 4）、后台 WiFi scans 过多：当手机不在充电状态而且应用在后台，每小时大于4次 WiFi scans。


### 3、CPU 时间片

**Android 手机保护 AP 和 BP 两个 CPU。AP 即 Application Processor，所有的用户界面以及 App 都是运行在 AP 上的。BP 即 Baseband Processor，手机射频都是运行在这个 CPU 上的。而一般我们所说的耗电，PowerProfile 文件里面的 CPU，指的是 AP**。

CPU 耗电通常有两种情况：

- 1）、**长期频繁唤醒：原本可以仅仅在 BP 上运行，消耗 5mA 左右，但是因为唤醒，AP 就会运作，不同手机情况不一样，至少会导致 20~30 mA 左右的耗电**。
- 2）、**CPU 长期高负荷：例如 App 退到后台的时候没有停止动画，或者程序有不退出的死循环等等，导致 CPU 满频、满核地跑**。


常用优化 CPU 时间片的方式有：

- 1）、**获取运行过程线程 CPU 消耗，定位 CPU 占用率异常方法**。
- 2）、**减少后台应用的主动运行**。


### 4、网络相关

通常情况下，使用 WIFI 连接网络时的功耗要低于使用移动网络的功耗。而使用移动网络传输数据，电量的消耗有以下3种状态：

- **Full power：高功率状态，移动网络连接被激活，允许设备以最大的传输速率进行操作**。
- **Low power：低功耗状态，对电量的消耗差不多是 Full power 状态下的 50%**。
- **Standby：空闲态，没有数据连接需要传输，电量消耗最少**。


因此，为了避免网络连接所带来的电量消耗，我们可以采用如下几种方案：

- 1）、尽量在 WIFI 环境下进行数据传输，在使用 WIFI 传输数据时，应该尽可能增大每个包的大小（不超过 MTU），并降低发包的频率。
- 2）、在蜂窝移动网络下需要对请求时机及次数控制：可以延迟执行的网络请求稍后一起发送，最好做到批量执行，尽量避免频繁的间隔网络请求，以尽量多地保持在 Radio Standby 状态。
- 3）、使用 JSON 和 Protobuf 进行数据压缩，减少时间。
- 4）、禁止使用轮询功能：轮询会导致网络请求一直处于被激活的状态，耗电过高。


### 5、定位相关

- 1）、**根据场景谨慎选择定位模式：对定位准确度没那么高的场景可以选择低精度模式**。
- 2）、**可以考虑网络定位代替 GPS**。
- 3）、**使用后务必及时关闭，减少更新频率，例如定位开启一定时间后超过某个阈值可以执行一个兜底策略：强制关闭 GPS**。


### 6、界面相关

- 1）、**离开界面后停止相关活动，例如关闭动画**。
- 2）、**耗电操作判断前后台，如果是后台则不执行相关操作**。


### 7、WakeLock 相关

WakeLock 常用于后台播放音视频、录制音视频、下载文件的情况。如果没有合理使用 WakeLock，则会造成严重的耗电问题，为了避免该问题，**我们应该定期针对使用了 WakeLock 的模块进行重点排查**。

我们可以使用 `adb shell dumpsys power` 命令查看系统当前的耗电信息，其中我们可以看到 WakeLock 列表，它通常会以 **”mLocks.size“ 或者 ”Wake Locks：size“** 开头。关于 WakeLock 的使用我们要着重注意以下几点：

- 1）、**注意成对使用 acquire、release**。
- 2）、**建议使用带参数的 acquire，避免没有及时释放而导致电量消耗过大**。
- 3）、**使用 finally 确保 release 一定会被调用**。
- 4）、**常亮场景使用 keepScreenOn 即可**。
- 5）、**WakeLock 有一个接口 setReferenceCounted，用来设置 WakeLock 的技术机制，官方默认为计数。true 为计数，false 为不计数。所谓计数即每一个 acquire 必须对应一个 release；不计数则是无论有多少个 acquire，一个 release 就可以释放。但是问题是有的第三方 ROM 它将默认设置为了不计数，以为我们需要在调用 newWakeLock 之后再调用 setReferenceCounted 为 false**。


### 8、计算优化

**浮点运算比整数运算更消耗 CPU 时间片，因此耗电也会增加**。避开浮点运算的优化方法如下所示：

- 1）、**除法变乘法**。
- 2）、**充分利用移位**。
- 3）、**在 native 层开发时，可以利用 ARM neon 指令集做并行运算，注意需要 ARM V7 及以上架构 CPU 才能支持**。


### 9、灭屏时停止动画

**我们可以监听灭屏以及亮屏的广播，在灭屏的时候停止 surfaceView 的动画绘制。在亮屏的时候，恢复动画的绘制**。


# 五、耗电监控

以后台耗电监控为主，必须监控的模块有：

- 1）、**Alarm wakeup**
- 2）、**WakeLock**
- 3）、**WiFi scans**
- 4）、**Network**


**必须监控的现场信息有** ：

- 1）、**堆栈信息**
- 2）、**是否充电**
- 3）、**电量水平**
- 4）、**应用前后台时间**
- 5）、**CPU 状态信息**


最后，我们需要 **提炼规则，将监控内容 => 抽象成规则**。

## 1、Java Hook

我们可以通过代理对应的 Service 实现，完成收集 Wakelock、Alarm、GPS 的申请堆栈、释放信息、手机充电状态等等。

[示例项目](https://github.com/simplezhli/Chapter19)


## 2、电量辅助监控实战

### 1）、获取运行时能耗文件

- 1）、adb pull /system/framework/framework-res.apk
- 2）、反编译，xml—》power_profile


### 2）、电量辅助监控

#### 线下使用 epic 进行 AOP 电量辅助统计

这里我们就以 WakeLock 的监控为例，切面代码如下所示：

```java
public static long sStartTime = 0;
@Insert(value = "acquire")
@TargetClass(value = "com.optimize.performance.wakelock.WakeLockUtils",scope = Scope.SELF)
public static void acquire(Context context){
    trace = Log.getStackTraceString(new Throwable());
    sStartTime = System.currentTimeMillis();
    Origin.callVoid();
    new Handler().postDelayed(new Runnable() {
        @Override
        public void run() {
            WakeLockUtils.release();
        }
    },1000);
}
@Insert(value = "release")
@TargetClass(value = "com.optimize.performance.wakelock.WakeLockUtils",scope = Scope.SELF)
public static void release(){
    LogUtils.i("PowerManager "+(System.currentTimeMillis() - sStartTime)+"/n"+trace);
```


此外，我们也可以利用 epic 来监控每个线程的执行时间，超过阈值则警告，示例代码如下所示：

```java
public static long runTime = 0;
@Insert(value = "run")
@TargetClass(value = "java.lang.Runnable",scope = Scope.ALL)
public void run(){
    runTime = System.currentTimeMillis();
    Origin.callVoid();
    LogUtils.i("runTime "+(System.currentTimeMillis() - runTime));
}
```


## 3、编译插桩

**写一个基础类，然后在统一的调用接口中添加监控逻辑**。这里我们可以参考 `Facebook Battery-Metrics` 获取、监控数据的方式。其代码如下所示：

```java
public class WakelockMetrics {

    /**
     * 获取 WakeLock
     *
     * @param wakeLock WakeLock
     * @param timeout 超时时间
     */
    public static void acquire(PowerManager.WakeLock wakeLock, long timeout) {
        wakeLock.acquire(timeout);
        // 监控 wakelock 相关信息
        Log.e("HOOOOOOOOK", "--acquireWakeLock--");
        Log.e("HOOOOOOOOK", Utils.getStackTrace());
        // 使用 Battery-Metrics 库统计其它维度的电量信息
        
    }

    /**
     * 释放 WakeLock
     *
     * @param wakeLock WakeLock
     */
    public static void release(PowerManager.WakeLock wakeLock) {
        wakeLock.release();
        Log.e("HOOOOOOOOK", "--releaseWakeLock--");
        Log.e("HOOOOOOOOK", Utils.getStackTrace());
        // 使用 Battery-Metrics 库统计其它维度的电量信息
        
    }

}
```


Gradle 耗电量统计插件中 BatteryCreateMethodVisitor 的核心实现代码如下所示：

```java
@Override
public void visitMethodInsn(int opcode, String owner, String name, String descriptor, boolean isInterface) {
    // 监控 Wakelock
    String monitorClass = "com/ss/android/ugc/bytex/example/battery_monitor/WakelockMetrics";
    if (!monitorClass.equals(className)
            && "android/os/PowerManager$WakeLock".equals(owner)
            && opcode == Opcodes.INVOKEVIRTUAL
            && "acquire".equals(name)) {
        mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                monitorClass,
                name,
                "(Landroid/os/PowerManager$WakeLock;J)V",
                isInterface
        );
        return;
    }
    if (!monitorClass.equals(className)
            && "android/os/PowerManager$WakeLock".equals(owner)
            && opcode == Opcodes.INVOKEVIRTUAL
            && "release".equals(name)) {
        mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                monitorClass,
                name,
                "(Landroid/os/PowerManager$WakeLock;)V",
                isInterface
        );
        return;
    }
    // 监控 Gps
    monitorClass = "com/ss/android/ugc/bytex/example/battery_monitor/GpsMetrics";
    if (!monitorClass.equals(className)
            && "android/location/LocationManager".equals(owner)
            && opcode == Opcodes.INVOKEVIRTUAL
            && "requestLocationUpdates".equals(name)) {
        mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                monitorClass,
                name,
                "(Landroid/location/LocationManager;Ljava/lang/String;JFLandroid/location/LocationListener;)V",
                isInterface
        );
        return;
    }
    if (!monitorClass.equals(className)
            && "android/location/LocationManager".equals(owner)
            && opcode == Opcodes.INVOKEVIRTUAL
            && "removeUpdates".equals(name)) {
        mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                monitorClass,
                name,
                "(Landroid/location/LocationManager;Landroid/location/LocationListener;)V",
                isInterface
        );
        return;
    }
    // 监控 Alarm Service
    monitorClass = "com/ss/android/ugc/bytex/example/battery_monitor/AlarmMetrics";
    if (!monitorClass.equals(className)
            && "android/app/AlarmManager".equals(owner)
            && opcode == Opcodes.INVOKEVIRTUAL
            && "set".equals(name)) {
        mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                monitorClass,
                name,
                "(Landroid/app/AlarmManager;IJLandroid/app/PendingIntent;)V",
                isInterface
        );
        return;
    }
    if (!monitorClass.equals(className)
            && "android/app/AlarmManager".equals(owner)
            && opcode == Opcodes.INVOKEVIRTUAL
            && "cancel".equals(name)) {
        mv.visitMethodInsn(
                Opcodes.INVOKESTATIC,
                monitorClass,
                name,
                "(Landroid/app/AlarmManager;Landroid/app/PendingIntent;)V",
                isInterface
        );
        return;
    }
    super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
}
```


#### 缺点

系统的代码插桩方案无法替换。


# 六、电量优化常见问题

## 1、怎么做电量测试？

电量相关的测试相对来说难度较大，因为 App 在具体手机上的耗电量无法准确统计，每一个手机所使用的硬件不一样，那么它相应的功耗就不一样。而且这个功耗值我们只能在线下通过导出手机的 power_profile.xml 文件拿到。

由于我们无法获取准确的耗电量，所以我们只能增加多个维度来辅助判断 App 是否耗电。

最后，我们可以分场景各个突破。

关于电量测试，我们可以针对各个功能场景进行针对性的专项测试。操作一段时间后，我们可以在手机设置—电量消耗里面，利用其数据作为判断依据。这样虽然直观，但精确度不行。

介绍 Battery Historian：

- Google 推出的一款 Android 电量分析工具，它支持 Android 5.0 及以上系统的电量分析。
- 它获取到的各个耗电模块的耗电信息要相对精确、丰富地多。例如 GPS、WaleLock、蓝牙 等的工作时间以及耗电量。
- 此外，它不仅可以针对单个 App 进行选择，也可以比对不同的电量场景的信息，比如 优化前、优化后 的信息。
- Battery Historian 的缺点在于它只能在线下使用。因此除了使用其在线下测试之外，我们还需要在线上增加一些电量的辅助监控，统计例如：耗电组件的使用次数、调用堆栈以及访问时间。这些都是与用户相关的基础电量消耗数据，如果有用户反馈，我们就可以通过这些信息来判断用户是不是有耗电的操作。


## 2、有哪些有效的电量优化手段？

因为我们不能在线上统计出 App 的电量消耗，因此需要在尽量保证 App 在正常使用下的耗电。对此我们采取了一系列的电量优化措施：

### 1）、网络相关

- 网络请求的时机以及次数，将可以延迟的网络请求批量发送，减少网络被激活的时机与次数。
- 此外，我们可以对网络传输数据进行压缩，以降低传输的时间与流量。
- 最后，一定要禁止使用轮询的方式来做业务操作。


### 2）、传感器相关

根据场景谨慎地选择传感器使用的模式，比如说在使用 GPS 的时候一般要避免使用高精度的模式，或者是尽量复用上一次的定位结果。

### 3）、WakeLock

我们在实际项目中使用 WakeLock 有几个注意事项，第一，acquire、release 要成对地释放，第二，尽量使用 acquire 的超时方法来设置超时时间，避免因为异常情况从而导致 WakeLock 而无法释放的情况，第三，关于 WakeLock 的释放一定要写在 try-catch-finally 的 finally 当中，保证 WakeLock 在异常情况下的释放。

### 4）、JobScheduler

JobScheduler 可以允许开发者在符合某些条件下创造执行在后台的任务，我们可以设置执行一些耗电操作的场景，比如说 处于 WIFI 状态下同时连接电源 的情况下。同时，要注意用户在离开界面后，要避免耗电的操作，比如说停止播放动画。通过这些操作，我们的 App 就不会比之前耗电了。


# 七、总结

对于电量优化来说，最重要的就是 **建立监控与自动化报警的一整套体系，只有发现了耗电的问题所在，才能使用针对性的解决措施**。


# 公众号

我的公众号 `JsonChao` 开通啦，欢迎关注~

![](https://user-gold-cdn.xitu.io/2020/6/11/172a29b8b626ef93?w=258&h=258&f=jpeg&s=28705)

# 参考链接：
---

- 1、《Android性能优化最佳实践》第六章 耗电优化（基础）
- 2、慕课网之Top团队大牛带你玩转Android性能分析与优化 第九章 App电量优化（进阶）
- 3、极客时间之Android开发高手课 耗电优化（进阶）
- 4、《Android移动性能实战》第五章 电池（经验）
- 5、[手机硬件已进入发展瓶颈，未来电池技术将如何突破？](https://mobile.pconline.com.cn/1089/10896724.html)
- 6、[VOOC 闪充](https://baike.baidu.com/item/VOOC%E9%97%AA%E5%85%85/13887450?fromtitle=%E5%85%85%E7%94%B55%E5%88%86%E9%92%9F%E9%80%9A%E8%AF%9D2%E5%B0%8F%E6%97%B6&fromid=18226496#reference-%5B4%5D-13589055-wrap)
- 7、[google/battery-historian](https://github.com/google/battery-historian)
- 8、[google/battery-historian-docs](https://github.com/facebookincubator/Battery-Metrics/blob/master/docs/references.md)
- 9、[google/battery-historian-api](https://facebookincubator.github.io/Battery-Metrics/)
- 10、[大众点评 App 的短视频耗电量优化实战](https://tech.meituan.com/2018/03/11/dianping-shortvideo-battery-testcase.html)
- 11、[Android 后台调度任务与省电](https://blog.dreamtobe.cn/2016/08/15/android_scheduler_and_battery/)
- 12、[Android P 电量管理](https://mp.weixin.qq.com/s/APhUH7MBDUZ6tQv0xDgaWQ)
- 13、[facebookincubator/Battery-Metrics](https://github.com/facebookincubator/Battery-Metrics)
- 14、[simplezhli/Chapter19](https://github.com/simplezhli/Chapter19)


# Contanct Me

##  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

##  ●  微信群：

> **由于微信群人数过多，麻烦大家想进微信群的朋友们，加我微信拉你进群。**
        

##  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


## About me

- ### Email: [chao.qu521@gmail.com]()
- ### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- ### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    

### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。



















