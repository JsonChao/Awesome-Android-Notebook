---

		title:  深入探索Android稳定性优化
		date: 2019/11/24 22:45:00   
		tags: 
		- 性能优化
		categories: 性能优化
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

众所周知，移动开发已经来到了后半场，为了能够在众多开发者中脱颖而出，我们需要对某一个领域有深入地研究与心得，对于Android开发者来说，目前，**有几个好的细分领域值得我们去建立自己的技术壁垒**，如下所示：

- 1、**性能优化专家**：具备深度性能优化与体系化APM建设的能力
- 2、**架构师**：具有丰富的应用架构设计经验与心得，对Android Framework层与热门三方库的实现原理与架构设计了如指掌。
- 3、**音视频/图像处理专家**：毫无疑问，掌握NDK，深入音视频与图像处理领域能让我们在未来几年大放异彩。
- 4、**大前端专家**：深入掌握Flutter及其设计原理与思想，可以让我们具有快速学习前端知识的能力。


在上述几个细分领域中，最难也最具技术壁垒的莫过于性能优化，要想成为一个顶尖的性能优化专家，**需要对许多领域的深度知识及广度知识有深入的了解与研究**，其中不乏**需要掌握架构师、NDK、Flutter所涉及的众多技能**。从这篇文章开始，笔者将会带领大家一步一步深入探索Android的性能优化。

为了能够全面地了解Android的性能优化知识体系，我们先看看我总结的下面这张图，如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ee14219183b4ff6964d3f9161cb9048~tplv-k3u1fbpfcp-zoom-1.image)


要做好应用的性能优化，我们需要建立一套成体系的性能优化方案，这套方案被业界称为 **APM（Application Performance Manange）**，为了让大家快速了解APM涉及的相关知识，笔者已经将其总结成图，如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51d4ebecf2f040ed81cf6966b8576fea~tplv-k3u1fbpfcp-zoom-1.image)


在建设APM和对App进行性能优化的过程中，我们**必须首先解决的是App的稳定性问题**，现在，**让我们搭乘航班，来深入探索Android稳定性优化的疆域**。


# 思维导图大纲

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62c0b9d18a3a4f53aa718042e4977678~tplv-k3u1fbpfcp-zoom-1.image)


# 目录

- 一、[正确认识](https://juejin.im/post/6844903972587716621#heading-2)
    - 1、稳定性纬度
    - 2、稳定性优化注意事项
    - 3、Crash 相关指标
    - 4、Crash 率评价
    - 5、Crash 关键问题
    - 6、APM Crash 部分整体架构
    - 7、责任归属
- 二、[Crash 优化](https://juejin.im/post/6844903972587716621#heading-19)
    - 1、单个 Crash 处理方案
    - 2、Crash 率治理方案
    - 3、Java Crash
    - 4、Java Crash 处理流程
    - 5、Native Crash
    - 6、疑难 Crash 解决方案
    - 7、进程保活
    - 8、总结
- 三、[ANR 优化](https://juejin.im/post/6844903972587716621#heading-101)
    - 1、ANR 监控实现方式
    - 2、ANR 优化
    - 3、关于 ANR 的一些常见问题
    - 4、理解 ANR 的触发流程
- 四、[移动端业务高可用方案建设](https://juejin.im/post/6844903972587716621#heading-133)
    - 1、业务高可用重要性
    - 2、业务高可用方案建设
    - 3、移动端容灾方案
- 五、[稳定性长效治理](https://juejin.im/post/6844903972587716621#heading-152)
    - 1、开发阶段
    - 2、测试阶段
    - 3、合码阶段
    - 4、发布阶段
    - 5、运维阶段
- 六、[稳定性优化问题](https://juejin.im/post/6844903972587716621#heading-158)
    - 1、你们做了哪些稳定性方面的优化？
    - 2、性能稳定性是怎么做的？
    - 3、业务稳定性如何保障？
    - 4、如果发生了异常情况，怎么快速止损？
- 七、[总结](https://juejin.im/post/6844903972587716621#heading-163)


# 一、正确认识

首先，我们必须对App的**稳定性**有正确的认识，它**是App质量构建体系中最基本和最关键的一环**。如果我们的App不稳定，并且经常不能正常地提供服务，那么用户大概率会卸载掉它。所以稳定性很重要，并且Crash是P0优先级，需要优先解决。
而且，稳定性可优化的面很广，它不仅仅只包含Crash这一部分，也包括卡顿、耗电等优化范畴。


## 1、稳定性纬度

应用的稳定性可以分为三个纬度，如下所示：
  
    
- 1、**Crash纬度**：最重要的指标就是应用的**Crash率**。  
- 2、**性能纬度**：包括启动速度、内存、绘制等等优化方向，相对于Crash来说是次要的，在做**应用性能体系化建设**之前，我们必须要确保应用的功能稳定可用。
- 3、**业务高可用纬度**：它是非常关键的一步，我们需要**采用多种手段来保证我们App的主流程以及核心路径的稳定性**，只有用户经常使用我们的App，它才有可能发现别的方面的问题。


## 2、稳定性优化注意事项

我们在做应用的稳定性优化的时候，需要注意三个要点，如下所示：


### 1、重在预防、监控必不可少

对于稳定性来说，如果App已经到了线上才发现异常，那其实已经造成了损失，所以，对于稳定性的优化，其重点在于预防。**从开发同学的编码环节，到测试同学的测试环节，以及到上线前的发布环节、上线后的运维环节，这些环节都需要来预防异常情况的发生**。如果异常真的发生了，也需要将想方设法将损失降到最低，争取用最小的代价来暴露尽可能多的问题。

此外，监控也是必不可少的一步，预防做的再好，到了线上，总会有各种各样的异常发生。所以，无论如何，我们都**需要有全面的监控手段来更加灵敏地发现问题**。


### 2、思考更深一层、重视隐含信息：如解决Crash问题时思考是否会引发同一类问题

当我们看到了一个Crash的时候，不能简单地只处理这一个Crash，而是需要思考更深一层，要考虑会不会在其它地方会有一样的Crash类型发生。如果有这样的情况，我们必须**对其统一处理和预防**。

此外，我们还要关注Crash相关的隐含信息，比如，在面试过程当中，面试官问你，你们应用的Crash率是多少，这个问题表明上问的是Crash率，但是实际上它是问你一些隐含信息的，**过高的Crash率就代表开发人员的水平不行，leader的架构能力不行，项目的各个阶段中优化的空间非常大**，这样一来，面试官对你的印象和评价也不会好。


### 3、长效保持需要科学流程

应用稳定性的建设过程是一个细活，所以很容易出现这个版本优化好了，但是在接下来的版本中如果我们不管它，它就会发生持续恶化的情况，因此，我们**必须从项目研发的每一个流程入手，建立科学完善的相关规范，才能保证长效的优化效果**。


## 3、Crash相关指标

要对应用的稳定性进行优化，我们就必须先了解与Crash相关的一些指标。


### 1、UV、PV 

- PV（Page View）：访问量
- UV（Unique Visitor）：独立访客，0 - 24小时内的同一终端只计算一次


### 2、UV、PV、启动、增量、存量 Crash率

- **UV Crash率（Crash UV / DAU）**：针对用户使用量的统计，统计一段时间内所有用户发生崩溃的占比，用于**评估Crash率的影响范围**，结合PV。需要注意的是，需要确保一直使用同一种衡量方式。
- **PV Crash率**：评估相关Crash影响的**严重程度**。
- **启动Crash率**：启动阶段，用户还没有完全打开App而发生的Crash，它是影响最严重的Crash，对用户伤害最大，无法通过热修复拯救，需结合客户端容灾，以进行App的自主修复。（这块后面会讲）
- **增量、存量Crash率**：增量Crash是指的**新增的Crash**，而存量Crash则表示一些**历史遗留bug**。增量Crash是新版本重点，存量Crash是需要持续啃的硬骨头，我们需要**优先解决增量、持续跟进存量问题**。


## 4、Crash率评价

那么，我们App的Crash率降低多少才能算是一个正常水平或优秀的水平呢？

- Java与Native的总崩溃率必须在千分之二以下。
- Crash率万分位为优秀：需要注意90%的Crash都是比较容易解决的，但是要解决最后的10%需要付出巨大的努力。


## 5、Crash关键问题

这里我们还需要关注Crash相关的关键问题，如果应用发生了Crash，我们应**该尽可能还原Crash现场**。因此，我们**需要全面地采集应用发生Crash时的相关信息**，如下所示：

- 堆栈、设备、OS版本、进程、线程名、Logcat
- 前后台、使用时长、App版本、小版本、渠道
- CPU架构、内存信息、线程数、资源包信息、用户行为日志


接着，采集完上述信息并上报到后台后，我们会在APM后台进行聚合展示，具体的展示信息如下所示：

- Crash现场信息
- Crash Top机型、OS版本、分布版本、区域
- Crash起始版本、上报趋势、是否新增、持续、量级


最后，我们可以**根据以上信息决定Crash是否需要立马解决以及在哪个版本进行解决**，关于APM聚合展示这块可以参考 **Bugly平台** 的APM后台聚合展示。
  
然后，我们再来看看与Crash相关的整体架构。


## 6、APM Crash部分整体架构

APM Crash部分的整体架构从上至下分为采集层、处理层、展示层、报警层。下面，我们来详细讲解一下每一层所做的处理。


### 1）、采集层

首先，我们需要在采集层这一层去获取足够多的Crash相关信息，以**确保能够精确定位到问题**。需要采集的信息主要为如下几种：

- **错误堆栈**
- **设备信息**
- **行为日志**
- **其它信息**


### 2）、处理层

然后，在处理层，我们会对App采集到的数据进行处理。

- **数据清洗**：将一些不符合条件的数据过滤掉，比如说，因为一些特殊情况，一些App采集到的数据不完整，或者由于上传数据失败而导致的数据不完整，这些数据在APM平台上肯定是无法全面地展示的，所以，首先我们需要把这些信息进行过滤。
- **数据聚合**：在这一层，我们会把Crash相关的数据进行聚合。
- **纬度分类**：如Top机型下的Crash、用户Crash率的前10%等等维度。
- **趋势对比**


### 3）、展示层

经过处理层之后，就会来到展示层，展示的信息为如下几类：

- **数据还原**
- **纬度信息**
- **起始版本**
- **其它信息**


### 4）、报警层

最后，就会来到报警层，当发生严重异常的时候，会通知相关的同学进行紧急处理。报警的规则我们可以自定义，例如**整体的Crash率，其环比（与上一期进行对比）或同比（如本月10号与上月10号）抖动超过5%，或者是单个Crash突然间激增**。报警的方式可以通过 **邮件、IM、电话、短信** 等等方式。


## 7、责任归属

最后，我们来看下Crash相关的非技术问题，需要注意的是，我们要解决的是如何长期保持较低的Crash率这个问题。我们**需要保证能够迅速找到相关bug的相关责任人并让开发同学能够及时地处理线上的bug**。具体的解决方法为如下几种：

- **设立专项小组轮值**：成立一个虚拟的专项小组，来专门跟踪每个版本线上的Crash率，**组内的成员可以轮流跟踪线上的Crash**，这样，就可以从源头来保证所有Crash一定会有人跟进。
- **自动匹配责任人**：**将APM平台与bug单系统打通，这样APM后台一旦发现紧急bug就能第一时间下发到bug单系统给相关责任人发提醒**。
- **处理流程全纪录**：我们需要**记录Crash处理流程的每一步，确保紧急Crash的处理不会被延误**。


# 二、Crash优化

## 1、单个Crash处理方案

对于单个Crash的处理方案我们可以按如下三个步骤来进行解决处理。

### 1）、根据堆栈及现场信息找答案

- 解决90%问题
- 解决完后需考虑产生Crash深层次的原因


### 2）、找共性：机型、OS、实验开关、资源包，考虑影响范围

### 3）、线下复现、远程调试


## 2、Crash率治理方案

要对应用的Crash率进行治理，一般需要对以下三种类型的Crash进行对应的处理，如下所示：

- 1）、解决线上常规Crash
- 2）、系统级Crash尝试Hook绕过
- 3）、疑难Crash重点突破或更换方案


## 3、Java Crash

出现未捕获异常，导致出现异常退出

    Thread.setDefaultUncaughtExceptionHandler()；


我们通过设置自定义的UncaughtExceptionHandler，就可以在崩溃发生的时候获取到现场信息。注意，**这个钩子是针对单个进程而言的，在多进程的APP中，监控哪个进程，就需要在哪个进程中设置一遍ExceptionHandler**。

获取主线程的堆栈信息：

    Looper.getMainLooper().getThread().getStackTrace();
    

获取当前线程的堆栈信息：

    Thread.currentThread().getStackTrace();
    

获取全部线程的堆栈信息：

    Thread.getAllStackTraces();
    
    
第三方Crash监控工具如 **Fabric、腾讯Bugly**，都是以**字符串拼接的方式将数组StackTraceElement[]转换成字符串形式**，进行保存、上报或者展示。


### 那么，我们如何反混淆上传的堆栈信息？

对此，我们一般有两种可选的处理方案，如下所示：

- 1、每次打包生成混淆APK的时候，需要把Mapping文件保存并上传到监控后台。
- 2、Android原生的反混淆的工具包是retrace.jar，在监控后台用来实时解析每个上报的崩溃时。它会将Mapping文件进行文本解析和对象实例化，这个过程比较耗时。因此可以将Mapping对象实例进行内存缓存，但为了防止内存泄露和内存过多占用，需要增加定期自动回收的逻辑。


### 如何获取logcat方法？

logcat日志流程是这样的，**应用层 --> liblog.so --> logd**，底层使用 **ring buffer** 来存储数据。获取的方式有以下三种：


#### 1、通过logcat命令获取。

- 优点：非常简单，兼容性好。
- 缺点：整个链路比较长，可控性差，失败率高，特别是堆破坏或者堆内存不足时，基本会失败。


#### 2、hook liblog.so实现

通过hook liblog.so 中的 **__android_log_buf_write** 方法，**将内容重定向到自己的buffer中**。

- 优点：简单，兼容性相对还好。
- 缺点：要一直打开。


#### 3、自定义获取代码。通过移植底层获取logcat的实现，通过socket直接跟logd交互。

- 优点：比较灵活，预先分配好资源，成功率也比较高。
- 缺点：实现非常复杂


### 如何获取Java 堆栈？

当发生native崩溃时，我们**通过unwind只能拿到Native堆栈**。但是我们**希望可以拿到当时各个线程的Java堆栈**。对于这个问题，目前有两种处理方式，分别如下所示：


#### 1、Thread.getAllStackTraces()。

##### 优点

简单，兼容性好。

##### 缺点

- 成功率不高，依靠系统接口在极端情况也会失败。
- 7.0之后这个接口是没有主线程堆栈。
- 使用Java层的接口需要暂停线程。


#### 2、hook libart.so。

通过hook **ThreadList和Thread** 的函数，**获得跟ANR一样的堆栈**。为了稳定性，**需要在fork的子进程中执行**。


- 优点：信息很全，基本跟ANR的日志一样，有native线程状态，锁信息等等。
- 缺点：黑科技的兼容性问题，失败时我们可以使用Thread.getAllStackTraces()兜底。


## 4、Java Crash处理流程

讲解了Java Crash相关的知识后，我们就可以去了解下Java Crash的处理流程，这里借用Gityuan流程图进行讲解，如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37f7f20b5dcd4005908ecdd796de76b7~tplv-k3u1fbpfcp-zoom-1.image)


#### 1、首先发生crash所在进程，在创建之初便准备好了defaultUncaughtHandler，用来来处理Uncaught Exception，并输出当前crash基本信息；

#### 2、调用当前进程中的AMP.handleApplicationCrash；经过binder ipc机制，传递到system_server进程；

#### 3、接下来，进入system_server进程，调用binder服务端执行AMS.handleApplicationCrash；

#### 4、从mProcessNames查找到目标进程的ProcessRecord对象；并将进程crash信息输出到目录/data/system/dropbox；

#### 5、执行makeAppCrashingLocked：

- 创建当前用户下的crash应用的error receiver，并忽略当前应用的广播；
- 停止当前进程中所有activity中的WMS的冻结屏幕消息，并执行相关一些屏幕相关操作；

#### 6、再执行handleAppCrashLocked方法：

- 当1分钟内同一进程连续crash两次时，且非persistent进程，则直接结束该应用所有activity，并杀死该进程以及同一个进程组下的所有进程。然后再恢复栈顶第一个非finishing状态的activity;
- 当1分钟内同一进程连续crash两次时，且persistent进程，，则只执行恢复栈顶第一个非finishing状态的activity;
- 当1分钟内同一进程未发生连续crash两次时，则执行结束栈顶正在运行activity的流程。

#### 7、通过mUiHandler发送消息SHOW_ERROR_MSG，弹出crash对话框；

#### 8、到此，system_server进程执行完成。回到crash进程开始执行杀掉当前进程的操作；

#### 9、当crash进程被杀，通过binder死亡通知，告知system_server进程来执行appDiedLocked()；

#### 10、最后，执行清理应用相关的activity/service/ContentProvider/receiver组件信息。


### 补充加油站：binder 死亡通知原理

这里我们还需要了解下binder 死亡通知的原理，其流程图如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0cf7299e8cf74b9ba20c8f534bede4ba~tplv-k3u1fbpfcp-zoom-1.image)


由于Crash进程中拥有一个Binder服务端ApplicationThread，而应用进程在创建过程调用attachApplicationLocked()，从而attach到system_server进程，在system_server进程内有一个ApplicationThreadProxy，这是相对应的Binder客户端。当Binder服务端ApplicationThread所在进程(即Crash进程)挂掉后，则Binder客户端能收到相应的死亡通知，从而进入binderDied流程。


## 5、Native Crash

特点:

- 访问非法地址
- 地址对齐出错
- 发送程序主动abort

上述都会产生相应的signal信号，导致程序异常退出。


### 1、合格的异常捕获组件

一个合格的异常捕获组件需要包含以下功能：

- 支持在crash时进行更多扩张操作
- 打印logcat和日志
- 上报crash次数
- 对不同crash做不同恢复措施
- 可以针对业务不断改进的适应


### 2、现有方案

#### 1、Google Breakpad

- 优点：权威、跨平台
- 缺点：代码体量较大


#### 2、Logcat

- 优点：利用安卓系统实现
- 缺点：需要在crash时启动新进程过滤logcat日志，不可靠


#### 3、coffeecatch

- 优点：实现简洁、改动容易
- 缺点：有兼容性问题


### 3、Native崩溃捕获流程

Native崩溃捕获的过程涉及到三端，这里我们分别来了解下其对应的处理。


#### 1、编译端

编译C/C++需将带符号信息的文件保留下来。

#### 2、客户端

捕获到崩溃时，将收集到尽可能多的有用信息写入日志文件，然后选择合适的时机上传到服务器。

#### 3、服务端

读取客户端上报的日志文件，寻找合适的符号文件，生成可读的C/C++调用栈。

### 4、Native崩溃捕获的难点

核心：如何确保客户端在各种极端情况下依然可以生成崩溃日志。

#### 1、文件句柄泄漏，导致创建日志文件失败？

提前申请文件句柄fd预留。


#### 2、栈溢出导致日志生成失败？

- 使用额外的栈空间signalstack，避免栈溢出导致进程没有空间创建调用栈执行处理函数。（signalstack：系统会在危险情况下把栈指针指向这个地方，使得可以在一个新的栈上运行信号处理函数）
- 特殊请求需直接替换当前栈，所以应在堆中预留部分空间。


#### 3、堆内存耗尽导致日志生产失败？

参考Breakpad重新封装Linux Syscall Support的做法以避免直接调用libc去分配堆内存。


#### 4、堆破坏或二次崩溃导致日志生成失败？

Breakpad使用了fork子进程甚至孙进程的方式去收集崩溃现场，即便出现二次崩溃，也只是这部分信息丢失。

这里说下Breakpad缺点：

- 生成的minidump文件时二进制的，包含过多不重要的信息，导致文件数MB。但minidump可以使用gdb调试、看到传入参数。

需要了解的是，未来Chromium会使用Crashpad替代Breakpad。


#### 5、想要遵循Android的文本格式并添加更多重要的信息？

改造Breakpad，增加Logcat信息，Java调用栈信息、其它有用信息。


### 5、Native崩溃捕获注册

一个Native Crash log信息如下：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a519cbcd0dd40f29ec6407104a04fbe~tplv-k3u1fbpfcp-zoom-1.image)

堆栈信息中 pc 后面跟的内存地址，就是当前函数的栈地址，我们可以通过下面的命令行得出出错的代码行数

    arm-linux-androideabi-addr2line -e 内存地址

下面列出全部的信号量以及所代表的含义：

    #define SIGHUP 1  // 终端连接结束时发出(不管正常或非正常)
    #define SIGINT 2  // 程序终止(例如Ctrl-C)
    #define SIGQUIT 3 // 程序退出(Ctrl-\)
    #define SIGILL 4 // 执行了非法指令，或者试图执行数据段，堆栈溢出
    #define SIGTRAP 5 // 断点时产生，由debugger使用
    #define SIGABRT 6 // 调用abort函数生成的信号，表示程序异常
    #define SIGIOT 6 // 同上，更全，IO异常也会发出
    #define SIGBUS 7 // 非法地址，包括内存地址对齐出错，比如访问一个4字节的整数, 但其地址不是4的倍数
    #define SIGFPE 8 // 计算错误，比如除0、溢出
    #define SIGKILL 9 // 强制结束程序，具有最高优先级，本信号不能被阻塞、处理和忽略
    #define SIGUSR1 10 // 未使用，保留
    #define SIGSEGV 11 // 非法内存操作，与 SIGBUS不同，他是对合法地址的非法访问，    比如访问没有读权限的内存，向没有写权限的地址写数据
    #define SIGUSR2 12 // 未使用，保留
    #define SIGPIPE 13 // 管道破裂，通常在进程间通信产生
    #define SIGALRM 14 // 定时信号,
    #define SIGTERM 15 // 结束程序，类似温和的 SIGKILL，可被阻塞和处理。通常程序如    果终止不了，才会尝试SIGKILL
    #define SIGSTKFLT 16  // 协处理器堆栈错误
    #define SIGCHLD 17 // 子进程结束时, 父进程会收到这个信号。
    #define SIGCONT 18 // 让一个停止的进程继续执行
    #define SIGSTOP 19 // 停止进程,本信号不能被阻塞,处理或忽略
    #define SIGTSTP 20 // 停止进程,但该信号可以被处理和忽略
    #define SIGTTIN 21 // 当后台作业要从用户终端读数据时, 该作业中的所有进程会收到SIGTTIN信号
    #define SIGTTOU 22 // 类似于SIGTTIN, 但在写终端时收到
    #define SIGURG 23 // 有紧急数据或out-of-band数据到达socket时产生
    #define SIGXCPU 24 // 超过CPU时间资源限制时发出
    #define SIGXFSZ 25 // 当进程企图扩大文件以至于超过文件大小资源限制
    #define SIGVTALRM 26 // 虚拟时钟信号. 类似于SIGALRM,     但是计算的是该进程占用的CPU时间.
    #define SIGPROF 27 // 类似于SIGALRM/SIGVTALRM, 但包括该进程用的CPU时间以及系统调用的时间
    #define SIGWINCH 28 // 窗口大小改变时发出
    #define SIGIO 29 // 文件描述符准备就绪, 可以开始进行输入/输出操作
    #define SIGPOLL SIGIO // 同上，别称
    #define SIGPWR 30 // 电源异常
    #define SIGSYS 31 // 非法的系统调用


一般关注SIGILL, SIGABRT, SIGBUS, SIGFPE, SIGSEGV, SIGSTKFLT, SIGSYS即可。

要订阅异常发生的信号，最简单的做法就是直接用一个循环遍历所有要订阅的信号，对每个信号调用sigaction()。

#### 注意

- JNI_OnLoad是最适合安装信号初识函数的地方。
- 建议在上报时调用Java层的方法统一上报。Native崩溃捕获注册。


### 6、崩溃分析流程

首先，应收集崩溃现场的一些相关信息，如下：

#### 1、崩溃信息

- 进程名、线程名
- 崩溃堆栈和类型
- 有时候也需要知道主线程的调用栈


#### 2、系统信息

- 系统运行日志


    /system/etc/event-log-tags

- 机型、系统、厂商、CPU、ABI、Linux版本等

注意，我们可以去寻找共性问题，如下：

- 设备状态
- 是否root
- 是否是模拟器


#### 3、内存信息

##### 系统剩余内存

    /proc/meminfo
    
当系统可用内存小于MemTotal的10%时，OOM、大量GC、系统频繁自杀拉起等问题非常容易出现。


##### 应用使用内存

包括Java内存、RSS、PSS

PSS和RSS通过/proc/self/smap计算，可以得到apk、dex、so等更详细的分类统计。


##### 虚拟内存

获取大小：

    /proc/self/status
   
    
获取其具体的分布情况：

    /proc/self/maps
    
    
需要注意的是，**对于32位进程，32位CPU，虚拟内存达到3GB就可能会引起内存失败的问题**。如果是64位的CPU，虚拟内存一般在3~4GB。如果支持64位进程，虚拟内存就不会成为问题。


#### 4、资源信息

如果应用堆内存和设备内存比较充足，但还出现内存分配失败，则可能跟资源泄漏有关。

##### 文件句柄fd

获取fd的限制数量：

    /proc/self/limits
    

一般**单个进程允许打开的最大句柄个数为1024**，如果**超过800需将所有fd和文件名输出日志进行排查**。


##### 线程数

获取线程数大小：

    /proc/self/status
    
一个线程一般占2MB的虚拟内存，线程数超过400个比较危险，需要将所有tid和线程名输出到日志进行排查。


##### JNI

容易出现引用失效、引用爆表等崩溃。

通过DumpReferenceTables统计JNI的引用表，进一步分析是否出现JNI泄漏等问题。


**补充加油站：dumpReferenceTables的出处**

在dalvik.system.VMDebug类中，是一个native方法，亦是static方法；在JNI中可以这么调用

    jclass vm_class = env->FindClass("dalvik/system/VMDebug");
    jmethodID dump_mid = env->GetStaticMethodID( vm_class, "dumpReferenceTables", "()V" );
    env->CallStaticVoidMethod( vm_class, dump_mid );
    

#### 5、应用信息

- 崩溃场景
- 关键操作路径
- 其它跟自身应用相关的自定义信息：运行时间、是否加载补丁、是否全新安装或升级。


#### 6、崩溃分析流程

接下来进行崩溃分析：

##### 1、确定重点

- 确认严重程度
- 优先解决Top崩溃或对业务有重大影响的崩溃：如启动、支付过程的崩溃
- Java崩溃：如果是OOM，需进一步查看日志中的内存信息和资源信息
- Native崩溃：查看signal、code、fault addr以及崩溃时的Java堆栈

常见的崩溃类型有：

- SIGSEGV：空指针、非法指针等
- SIGABRT：ANR、调用abort推出等

如果是ANR，先看主线程堆栈、是否因为锁等待导致，然后看ANR日志中的iowait、CPU、GC、systemserver等信息，确定是I/O问题或CPU竞争问题还是大量GC导致的ANR。

注意，当从一条崩溃日志中无法看出问题原因时，需要查看相同崩溃点下的更多崩溃日志，或者也可以查看内存信息、资源信息等进行异常排查。


##### 2、查找共性

机型、系统、ROM、厂商、ABI这些信息都可以作为共性参考，对于下一步复现问题有明确指引。


##### 3、尝试复现

复现之后再增加日志或使用Debugger、GDB进行调试。如不能复现，可以采用一些高级手段，如xlog日志、远程诊断、动态分析等等。


补充加油站：系统崩溃解决方式

- 1、通过共性信息查找可能的原因
- 2、尝试使用其它使用方式规避
- 3、Hook解决


### 7、实战：使用Breakpad捕获native崩溃

首先，这里给出《Android开发高手课》张绍文老师写的[crash捕获示例工程](https://github.com/AndroidAdvanceWithGeektime/Chapter01)，工程里面已经集成了Breakpad 来获取发生 native crash 时候的系统信息和线程堆栈信息。下面来详细介绍下使用Breakpad来分析native崩溃的流程：

#### 1、示例工程是采用cmake的构建方式，所以需要先到Android Studio中SDK Manager中的SDK Tools下下载NDK和cmake。

#### 2、安装实例工程后，点击CRASH按钮产生一个native崩溃。生成的 crash信息，如果授予Sdcard权限会优先存放在/sdcard/crashDump下，便于我们做进一步的分析。反之会放到目录 /data/data/com.dodola.breakpad/files/crashDump中。

#### 3、使用adb pull命令将抓取到的crash日志文件放到电脑本地目录中：

    adb pull /sdcard/crashDump/***.dmp > ~/Documents/crash_log.dmp
    
#### 4、下载并编译Breakpad源码，在src/processor目录下找到minidump_stackwalk，使用这个工具将dmp文件转换为txt文件：

    // 在项目目录下clone Breakpad仓库
    git clone https://github.com/google/breakpad.git
   
    // 切换到Breakpad根目录进行配置、编译
    cd breakpad
    ./configure && make

    // 使用src/processor目录下的minidump_stackwalk工具将dmp文件转换为txt文件
    ./src/processor/minidump_stackwalk ~/Documents/crashDump/crash_log.dmp >crash_log.txt 

#### 5、打开crash_log.txt，可以得到如下内容：

    Operating system: Android
                      0.0.0 Linux 4.4.78-perf-g539ee70 #1 SMP PREEMPT Mon Jan 14 17:08:14 CST 2019 aarch64
    CPU: arm64
         8 CPUs
    
    GPU: UNKNOWN
    
    Crash reason:  SIGSEGV /SEGV_MAPERR
    Crash address: 0x0
    Process uptime: not available
    
    Thread 0 (crashed)
     0  libcrash-lib.so + 0x650


其中我们需要的关键信息为CPU是arm64的，并且crash的地址为0x650。接下来我们需要将这个地址转换为代码中对应的行。

#### 6、使用ndk 中提供的addr2line来根据地址进行一个符号反解的过程。

如果是arm64的so使用 $NDKHOME/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/aarch64-linux-android-addr2line。

如果是arm的so使用 $NDKHOME/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-addr2line。

由crash_log.txt的信息可知，我们机器的cpu架构是arm64的，因此需要使用aarch64-linux-android-addr2line这个命令行工具。该命令的一般使用格式如下：
    // 注意：在mac下 ./ 代表执行文件    ./aarch64-linux-android-addr2line -e 对应的.so 需要解析的地址
    
    
上述中对应的.so文件在项目编译之后，会出现在Chapter01-master/sample/build/intermediates/merged_native_libs/debug/out/lib/arm64-v8a/libcrash-lib.so这个位置，由于我的手机CPU架构是arm64的，所以这里选择的是arm64-v8a中的libcrash-lib.so。接下来我们使用aarch64-linux-android-addr2line这个命令：

    ./aarch64-linux-android-addr2line -f -C -e ~/Documents/open-project/Chapter01-master/sample/build/intermediates/merged_native_libs/debug/out/lib/arm64-v8a/libcrash-lib.so 0x650
    
    参数含义：
    -e --exe=<executable>：指定需要转换地址的可执行文件名。
    -f --functions：在显示文件名、行号输出信息的同时显示函数名信息。
    -C --demangle[=style]：将低级别的符号名解码为用户级别的名字。
    
    
结果输出为：

    Crash()
    /Users/quchao/Documents/open-project/Chapter01-master/sample/src/main/cpp/crash.cpp:10


由此，我们得出crash的代码行为crahs.cpp文件中的第10行，接下来根据项目具体情况进行相应的修改即可。

Tips：这是从事NDK开发（音视频、图像处理、OpenCv、热修复框架开发）同学调试native层错误时经常要使用的技巧，强烈建议熟练掌握。


## 6、疑难Crash解决方案

最后，笔者这里再讲解下一些疑难Crash的解决方案。

### 问题1：如何解决Android 7.0 Toast BadTokenException？

参考Android 8.0 try catch的做法，代理Toast里的mTN（handler）就可以实现捕获异常。


### 问题2：如何解决 SharedPreference apply 引起的 ANR 问题

#### apply为什么会引起ANR？

SP 调用 apply 方法，会创建一个等待锁放到 QueuedWork 中，并将真正数据持久化封装成一个任务放到异步队列中执行，任务执行结束会释放锁。Activity onStop 以及 Service 处理 onStop，onStartCommand 时，执行 QueuedWork.waitToFinish() 等待所有的等待锁释放。


#### 如何解决？

所有此类 ANR 都是经由 QueuedWork.waitToFinish() 触发的，只要在调用此函数之前，将其中保存的队列手动清空即可。

具体是Hook ActivityThrad的Handler变量，拿到此变量后给其设置一个Callback，Handler 的 dispatchMessage 中会先处理 callback。最后在 Callback
中调用队列的清理工作，注意队列清理需要反射调用 QueuedWork。

#### 注意

apply 机制本身的失败率就比较高（1.8%左右），清理等待锁队列对持久化造成的影响不大。


### 问题3：如何解决TimeoutExceptin异常？

它是由系统的FinalizerWatchdogDaemon抛出来的。

这里首先介绍下看门狗 WatchDog，它 的作用是监控重要服务的运行状态，当重要服务停止时，发生 Timeout 异常崩溃，WatchDog 负责将应用重启。而当关闭 WatchDog（执行stop（）方法）后，当重要服务停止时，也不会发生 Timeout 异常，是一种通过非正常手段防止异常发生的方法。


#### 规避方案

stop方法，在Android 6.0之前会有线程同步问题。
因为6.0之前调用threadToStop的interrupt方法是没有加锁的，所以可能会有线程同步的问题。
 
注意：Stop的时候有一定概率导致即使没有超时也会报timeoutexception。

#### 缺点

只是为了避免上报异常采取的一种hack方案，并没有真正解决引起finialize超时的问题。


### 问题4：如何解决输入法的内存泄漏？

通过反射将输入法的两个View置空。


## 7、进程保活

我们可以**利用SyncAdapter提高进程优先级**，它是Android系统提供一个**账号同步机制**，它属于**核心进程级别**，而使用了SyncAdapter的进程优先级本身也会提高，使用方式请Google，关联SyncAdapter后，进程的**优先级变为1**，仅低于前台正在运行的进程，因此可以**降低应用被系统杀掉的概率**。


## 8、总结

对于App的Crash优化，总的来说，我们需要考虑以下四个要点：

- 1、重在预防：重视应用的整个流程、包括开发人员的培训、编译检查、静态扫描、规范的测试、灰度、发布流程等
- 2、不应该随意使用try catch去隐藏问题：而应该从源头入手，了解崩溃的本质原因，保证后面的运行流程。
- 3、解决崩溃的过程应该由点到面，考虑一类崩溃怎么解决。
- 4、崩溃与内存、卡顿、I/O内存紧密相关


# 三、ANR优化

## 1、ANR监控实现方式

### 1、使用FileObserver监听 /data/anr/traces.txt的变化

#### 缺点

高版本ROM需要root权限。

#### 解决方案

海外Google Play服务、国内Hardcoder。


### 2、监控消息队列的运行时间（WatchDog）

#### 卡顿监控原理：

利用主线程的消息队列处理机制，应用发生卡顿，一定是在dispatchMessage中执行了耗时操作。我们通过给主线程的Looper设置一个Printer，打点统计dispatchMessage方法执行的时间，如果超出阀值，表示发生卡顿，则dump出各种信息，提供开发者分析性能瓶颈。

为卡顿监控代码增加ANR的线程监控，在发送消息时，在ANR线程中保存一个状态，主线程消息执行完后再Reset标志位。如果在ANR线程中收到发送消息后，超过一定时间没有复位，就可以任务发生了ANR。

#### 缺点

- 无法准确判断是否真正出现ANR，只能说明APP发生了UI阻塞，需要进行二次校验。校验的方式就是等待手机系统出现发生了Error的进程，并且Error类型是NOT_RESPONDING（值为2）。
在每次出现ANR弹框前，Native层都会发出signal为SIGNAL_QUIT（值为3）的信号事件，也可以监听此信号。
- 无法得到完整ANR日志
- 隶属于卡顿优化的方式


### 3、需要考虑应用退出场景

- 主动自杀
- Process.killProcess()、exit()等。
- 崩溃
- 系统重启
- 系统异常、断电、用户重启等：通过比较应用开机运行时间是否比之前记录的值更小。
- 被系统杀死
- 被LMK杀死、从系统的任务管理器中划掉等。

#### 注意

由于traces.txt上传比较耗时，所以一般线下采用，线上建议综合ProcessErrorStateInfo和出现ANR时的堆栈信息来实现ANR的实时上传。


## 2、ANR优化

ANR发生原因：没有在规定的时间内完成要完成的事情。


### ANR分类

#### 发生场景

- Activity onCreate方法或Input事件超过5s没有完成
- BroadcastReceiver前台10s，后台60s
- ContentProvider 在publish过超时10s;
- Service前台20s，后台200s


#### 发生原因

- 主线程有耗时操作
- 复杂布局
- IO操作
- 被子线程同步锁block
- 被Binder对端block
- Binder被占满导致主线程无法和SystemServer通信
- 得不到系统资源（CPU/RAM/IO）


从进程角度看发生原因有：

- 当前进程：主线程本身耗时或者主线程的消息队列存在耗时操作、主线程被本进程的其它子线程所blocked
- 远端进程：binder call、socket通信


Andorid系统监测ANR的核心原理是消息调度和超时处理。


### ANR排查流程

#### 1、Log获取

1、抓取bugreport

    adb shell bugreport > bugreport.txt

2、直接导出/data/anr/traces.txt文件

    adb pull /data/anr/traces.txt trace.txt


#### 2、搜索“ANR in”处log关键点解读

- 发生时间（可能会延时10-20s）
- pid：当pid=0，说明在ANR之前，进程就被LMK杀死或出现了Crash，所以无法接受到系统的广播或者按键消息，因此会出现ANR
- cpu负载Load: 7.58 / 6.21 / 4.83

    代表此时一分钟有平均有7.58个进程在等待
    1、5、15分钟内系统的平均负荷
    当系统负荷持续大于1.0，必须将值降下来
    当系统负荷达到5.0，表面系统有很严重的问题

- cpu使用率


    CPU usage from 18101ms to 0ms ago
    28% 2085/system_server: 18% user + 10% kernel / faults: 8689 minor 24 major
    11% 752/android.hardware.sensors@1.0-service: 4% user + 6.9% kernel / faults: 2 minor
    9.8% 780/surfaceflinger: 6.2% user + 3.5% kernel / faults: 143 minor 4 major

上述表示Top进程的cpu占用情况。

##### 注意

如果CPU使用量很少，说明主线程可能阻塞。


#### 3、在bugreport.txt中根据pid和发生时间搜索到阻塞的log处

    ----- pid 10494 at 2019-11-18 15:28:29 -----


#### 4、往下翻找到“main”线程则可看到对应的阻塞log

    "main" prio=5 tid=1 Sleeping
    | group="main" sCount=1 dsCount=0 flags=1 obj=0x746bf7f0 self=0xe7c8f000
    | sysTid=10494 nice=-4 cgrp=default sched=0/0 handle=0xeb6784a4
    | state=S schedstat=( 5119636327 325064933 4204 ) utm=460 stm=51 core=4 HZ=100
    | stack=0xff575000-0xff577000 stackSize=8MB
    | held mutexes=


上述关键字段的含义如下所示：


- tid：线程号
- sysTid：主进程线程号和进程号相同
- Waiting/Sleeping：各种线程状态
- nice：nice值越小，则优先级越高，-17~16
- schedstat：Running、Runable时间(ns)与Switch次数
- utm：该线程在用户态的执行时间(jiffies)
- stm：该线程在内核态的执行时间(jiffies)
- sCount：该线程被挂起的次数
- dsCount：该线程被调试器挂起的次数
- self：线程本身的地址


#### 补充加油站：各种线程状态

需要注意的是，这里的各种线程状态指的是Native层的线程状态，关于Java线程状态与Native线程状态的对应关系如下所示：

    
    enum ThreadState {
      //                                   Thread.State   JDWP state
      kTerminated = 66,                 // TERMINATED     TS_ZOMBIE    Thread.run has returned, but Thread* still around
      kRunnable,                        // RUNNABLE       TS_RUNNING   runnable
      kTimedWaiting,                    // TIMED_WAITING  TS_WAIT      in Object.wait() with a timeout
      kSleeping,                        // TIMED_WAITING  TS_SLEEPING  in Thread.sleep()
      kBlocked,                         // BLOCKED        TS_MONITOR   blocked on a monitor
      kWaiting,                         // WAITING        TS_WAIT      in Object.wait()
      kWaitingForLockInflation,         // WAITING        TS_WAIT      blocked inflating a thin-lock
      kWaitingForTaskProcessor,         // WAITING        TS_WAIT      blocked waiting for taskProcessor
      kWaitingForGcToComplete,          // WAITING        TS_WAIT      blocked waiting for GC
      kWaitingForCheckPointsToRun,      // WAITING        TS_WAIT      GC waiting for checkpoints to run
      kWaitingPerformingGc,             // WAITING        TS_WAIT      performing GC
      kWaitingForDebuggerSend,          // WAITING        TS_WAIT      blocked waiting for events to be sent
      kWaitingForDebuggerToAttach,      // WAITING        TS_WAIT      blocked waiting for debugger to attach
      kWaitingInMainDebuggerLoop,       // WAITING        TS_WAIT      blocking/reading/processing debugger events
      kWaitingForDebuggerSuspension,    // WAITING        TS_WAIT      waiting for debugger suspend all
      kWaitingForJniOnLoad,             // WAITING        TS_WAIT      waiting for execution of dlopen and JNI on load code
      kWaitingForSignalCatcherOutput,   // WAITING        TS_WAIT      waiting for signal catcher IO to complete
      kWaitingInMainSignalCatcherLoop,  // WAITING        TS_WAIT      blocking/reading/processing signals
      kWaitingForDeoptimization,        // WAITING        TS_WAIT      waiting for deoptimization suspend all
      kWaitingForMethodTracingStart,    // WAITING        TS_WAIT      waiting for method tracing to start
      kWaitingForVisitObjects,          // WAITING        TS_WAIT      waiting for visiting objects
      kWaitingForGetObjectsAllocated,   // WAITING        TS_WAIT      waiting for getting the number of allocated objects
      kWaitingWeakGcRootRead,           // WAITING        TS_WAIT      waiting on the GC to read a weak root
      kWaitingForGcThreadFlip,          // WAITING        TS_WAIT      waiting on the GC thread flip (CC collector) to finish
      kStarting,                        // NEW            TS_WAIT      native thread started, not yet ready to run managed code
      kNative,                          // RUNNABLE       TS_RUNNING   running in a JNI native method
      kSuspended,                       // RUNNABLE       TS_RUNNING   suspended by GC or debugger
    };
    

#### 其它分析方法：Java线程调用分析方法

- 先使用jps命令列出当前系统中运行的所有Java虚拟机进程，拿到应用进程的pid。
- 然后再使用jstack命令查看该进程中所有线程的状态以及调用关系，以及一些简单的分析结果。


## 3、关于ANR的一些常见问题

### 1、sp调用apply导致anr问题？

虽然apply并不会阻塞主线程，但是会将等待时间转嫁到主线程。

### 2、检测运行期间是否发生过异常退出？

在应用启动时设定一个标志，在主动自杀或崩溃后更新标志 ，下次启动时检测此标志即可判断。


## 4、理解ANR的触发流程

broadcast跟service超时机制大抵相同，但有一个非常隐蔽的技能点，那就是通过静态注册的广播超时会受SharedPreferences(简称SP)的影响。

当SP有未同步到磁盘的工作，则需等待其完成，才告知系统已完成该广播。并且只有XML静态注册的广播超时检测过程会考虑是否有SP尚未完成，动态广播并不受其影响。

- 对于Service, Broadcast, Input发生ANR之后,最终都会调用AMS.appNotResponding。
- 对于provider,在其进程启动时publish过程可能会出现ANR, 则会直接杀进程以及清理相应信息,而不会弹出ANR的对话框。
- 对于输入事件发生ANR，首先会调用InputMonitor.notifyANR，最终也会调用AMS.appNotResponding。


### 1、AMS.appNotResponding流程

- 输出ANR Reason信息到EventLog. 也就是说ANR触发的时间点最接近的就是EventLog中输出的am_anr信息。
- 收集并输出重要进程列表中的各个线程的traces信息，该方法较耗时。
- 输出当前各个进程的CPU使用情况以及CPU负载情况。
- 将traces文件和 CPU使用情况信息保存到dropbox，即data/system/dropbox目录（ANR信息最为重要的信息）。
- 根据进程类型,来决定直接后台杀掉,还是弹框告知用户。


### 2、AMS.dumpStackTraces流程

#### 1、收集firstPids进程的stacks：

- 第一个是发生ANR进程；
- 第二个是system_server；
- 其余的是mLruProcesses中所有的persistent进程。

#### 2、收集Native进程的stacks。(dumpNativeBacktraceToFile)

- 依次是mediaserver,sdcard,surfaceflinger进程。

#### 3、收集lastPids进程的stacks：

- 依次输出CPU使用率top 5的进程；

##### 注意

上述导出每个进程trace时，进程之间会休眠200ms。


# 四、移动端业务高可用方案建设

## 1、业务高可用重要性

关于业务高可用重要性有如下五点：

- 高可用
- 性能
- 业务
- 侧重于用户功能完整可用
- 真实影响收入


## 2、业务高可用方案建设

业务高可用方案建设需要注意的点比较繁杂，但是总体可以归结为如下几点：

- 数据采集
- 梳理项目主流程、核心路径、关键节点
- Aop自动采集、统一上报
- 报警策略：阈值报警、趋势报警、特定指标报警、直接上报（或底阈值）
- 异常监控
- 单点追查：需要针对性分析的特定问题，全量日志回捞，专项分析
- 兜底策略
- 配置中心、功能开关
- 跳转分发中心（组件化路由）


## 3、移动端容灾方案

### 灾包括：

- 性能异常
- 业务异常

### 传统流程：

用户反馈、重新打包、渠道更新、不可接受。


### 容灾方案建设

关于容灾方案的建设主要可以细分为以下七点，下面，我们分别来了解下。


#### 1、功能开关

配置中心，服务端下发配置控制


##### 针对场景

- 功能新增
- 代码改动


#### 2、统跳中心

- 界面切换通过路由，路由决定是否重定向
- Native Bug不能热修复则跳转到临时H5页面


#### 3、动态化修复

热修复能力，可监控、灰度、回滚、清除。


#### 4、推拉结合、多场景调用保证到达率

#### 5、Weex、RN增量更新

#### 6、安全模式

微信读书、蘑菇街、淘宝、天猫等“重运营”的APP都使用了安全模式保障客户端启动流程，启动失败后给用户自救机会。先介绍一下它的核心特点：

- 根据Crash信息自动恢复，多次启动失败重置应用为安装初始状态
- 严重Bug可阻塞性热修复


##### 安全模式设计

配置后台：统一的配置后台，具备灰度发布机制

1、客户端能力：

- 在APP连续Crash的情况下具备分级、无感自修复能力
- 具备同步热修复能力
- 具备指定触发某项特定功能的能力
- 具体功能注册能力，方便后期扩展安全模式

2、数据统计及告警

- 统一的数据平台
- 监控告警功能，及时发现问题
- 查看热修复成功率等数据

3、快速测试

- 优化预发布环境下测试
- 优化回归验证安全模式难点等


##### 天猫安全模式原理

1、如何判断异常退出？

APP启动时记录一个flag值，满足以下条件时，将flag值清空

- APP正常启动10秒
- 用户正常退出应用
- 用户主动从前台切换到后台

如果在启动阶段发生异常，则flag值不会清空，通过flag值就可以判断客户端是否异常退出，每次异常退出，flag值都+1。

2、安全模式的分级执行策略

分为两级安全模式，连续Crash 2次为一级安全模式，连续Crash 2次及以上为二级安全模式。

业务线可以在一级安全模式中注册行为，比如清空缓存数据，再进入该模式时，会使用注册行为尝试修复客户端
如果一级安全模式无法修复APP，则进入二级安全模式将APP恢复到初次安装状态，并将Document、Library、Cache三个根目录清空。

3、热修复执行策略

只要发现配置中需要热修复，APP就会同步阻塞进行热修复,保证修复的及时性

4、灰度方案

灰度时，配置中会包含灰度、正式两份配置及其灰度概率
APP根据特定算法算出自己是否满足灰度条件，则使用灰度配置


##### 易用性考量

1、接入成本

完善文档、接口简洁

2、统一配置后台

可按照APP、版本配置

3、定制性

支持定制功能，让接入方来决定具体行为

4、灰度机制

5、数据分析

采用统一数据平台，为安全模式改进提供依据

6、快速测试

创建更多的针对性测试案例，如模拟连续Crash


#### 7、异常熔断

当多次请求失败则可让网络库主动拒绝请求。


### 容灾方案集合路径

功能开关 -> 统跳中心 -> 动态修复 -> 安全模式


# 五、稳定性长效治理

要实现App稳定性的长效治理，我们需要从 **开发阶段
=> 测试阶段 => 合码阶段 => 发布阶段 => 运维阶段** 这五个阶段来做针对性地处理。
 

## 1、开发阶段

- 统一编码规范、增强编码功底、技术评审、CodeReview机制
- 架构优化
- 能力收敛
- 统一容错：如在网络库utils中统一对返回信息进行预校验，如不合法就直接不走接下来的流程。


## 2、测试阶段

- 功能测试、自动化测试、回归测试、覆盖安装
- 特殊场景、机型等边界测试：如服务端返回异常数据、服务端宕机
- 云测平台：提供更全面的机型进行测试


## 3、合码阶段

- 编译检测、静态扫描
- 预编译流程、主流程自动回归


## 4、发布阶段

- 多轮灰度
- 分场景、纬度全面覆盖


## 5、运维阶段

- 灵敏监控
- 回滚、降级策略
- 热修复、本地容灾方案


# 六、稳定性优化问题

## 1、你们做了哪些稳定性方面的优化？

随着项目的逐渐成熟，用户基数逐渐增多，DAU持续升高，我们遇到了很多稳定性方面的问题，对于我们技术同学遇到了很多的挑战，用户经常使用我们的App卡顿或者是功能不可用，因此我们就针对稳定性开启了专项的优化，我们主要优化了三项：


- **Crash专项优化**
- **性能稳定性优化**
- **业务稳定性优化**


通过这三方面的优化我们搭建了移动端的高可用平台。同时，也做了很多的措施来让App真正地实现了高可用。 


## 2、性能稳定性是怎么做的？

- **全面的性能优化**：启动速度、内存优化、绘制优化
- **线下发现问题、优化为主**
- **线上监控为主**
- **Crash专项优化**


我们针对启动速度，内存、布局加载、卡顿、瘦身、流量、电量等多个方面做了多维的优化。

我们的优化主要分为了两个层次，即线上和线下，针对于线下呢，我们侧重于发现问题，直接解决，将问题尽可能在上线之前解决为目的。而真正到了线上呢，我们最主要的目的就是为了监控，对于各个性能纬度的监控呢，可以让我们尽可能早地获取到异常情况的报警。

同时呢，对于线上最严重的性能问题性问题：Crash，我们做了专项的优化，不仅优化了Crash的具体指标，而且也尽可能地获取了Crash发生时的详细信息，结合后端的聚合、报警等功能，便于我们快速地定位问题。


## 3、业务稳定性如何保障？

- **数据采集 + 报警**
- 需要对项目的**主流程与核心路径进行埋点监控**，
- 同时还**需知道每一步发生了多少异常**，这样，我们就知道了**所有业务流程的转换率以及相应界面的转换率**
- **结合大盘，如果转换率低于某个值，进行报警**
- **异常监控 + 单点追查**
- **兜底策略，如天猫安全模式**


移动端业务高可用它侧重于用户功能完整可用，主要是为了解决一些线上一些异常情况导致用户他虽然没有崩溃，也没有性能问题，但是呢，只是单纯的功能不可用的情况，我们需要对项目的主流程、核心路径进行埋点监控，来计算每一步它真实的转换率是多少，同时呢，还需要知道在每一步到底发生了多少异常。这样我们就知道了所有业务流程的转换率以及相应界面的转换率，有了大盘的数据呢，我们就知道了，如果转换率或者是某些监控的成功率低于某个值，那很有可能就是出现了线上异常，结合了相应的报警功能，我们就不需要等用户来反馈了，这个就是业务稳定性保障的基础。

同时呢，对于一些特殊情况，比如说，开发过程当中或代码中出现了一些catch代码块，捕获住了异常，让程序不崩溃，这其实是不合理的，程序虽然没有崩溃，当时程序的功能已经变得不可用，所以呢，这些被catch的异常我们也需要上报上来，这样我们才能知道用户到底出现了什么问题而导致的异常。此外，线上还有一些单点问题，比如说用户点击登录一直进不去，这种就属于单点问题，其实我们是无法找出其和其它问题的共性之处的，所以呢，我们就必须要找到它对应的详细信息。

最后，如果发生了异常情况，我们还采取了一系列措施进行快速止损。（=>4）


## 4、如果发生了异常情况，怎么快速止损？

- **功能开关**
- **统跳中心**
- **动态修复：热修复、资源包更新**
- **自主修复：安全模式**


首先，需要让App具备一些高级的能力，我们对于任何要上线的新功能，要加上一个功能的开关，通过配置中心下发的开关呢，来决定是否要显示新功能的入口。如果有异常情况，可以紧急关闭新功能的入口，那就可以让这个App处于可控的状态了。

然后，我们需要给App设立路由跳转，所有的界面跳转都需要通过路由来分发，如果我们匹配到需要跳转到有bug的这样一个新功能时，那我们就不跳转了，或者是跳转到统一的异常正处理中的界面。如果这两种方式都不可以，那就可以考虑通过热修复的方式来动态修复，目前热修复的方案其实已经比较成熟了，我们完全可以低成本地在我们的项目中添加热修复的能力，当然，如果有些功能是由RN或WeeX来实现就更好了，那就可以通过更新资源包的方式来实现动态更新。而这些如果都不可以的话呢，那就可以考虑自己去给应用加上一个自主修复的能力，如果App启动多次的话，那就可以考虑清空所有的缓存数据，将App重置到安装的状态，到了最严重的等级呢，可以阻塞主线程，此时一定要等App热修复成功之后才允许用户进入。


# 七、总结

Android稳定性优化是一个需要 **长期投入，持续运营和维护** 的一个过程，上文中我们不仅深入探讨了Java Crash、Native Crash和ANR的解决流程及方案，还分析了其内部实现原理和监控流程。到这里，可以看到，要想做好稳定性优化，我们 **必须对虚拟机运行、Linux信号处理和内存分配** 有一定程度的了解，**只有深入了解这些底层知识，我们才能比别人设计出更好的稳定性优化方案**。


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90c3dbed66a545d9b64a9fe9d6cba365~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、《Android性能优化最佳实践》第五章 稳定性优化 

2、[慕课网之国内Top团队大牛带你玩转Android性能分析与优化 第十一章 App稳定性优化](https://coding.imooc.com/class/308.html)

3、[极客时间之Android开发高手课 崩溃优化](https://time.geekbang.org/column/article/70966) 

4、[Android 平台 Native 代码的崩溃捕获机制及实现](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w?)

5、[安全模式：天猫App启动保护实践](https://mp.weixin.qq.com/s__biz=MzUxMzcxMzE5Ng==&mid=2247488429&idx=1&sn=448b414a0424d06855359b3eb2ba8569&source=41#wechat_redirect)

6、[美团外卖Android Crash治理之路](https://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651748107&idx=1&sn=55dff1b286e92cfb6aaee776df8ec89e&chksm=bd12ae468a652750a7624c30eca56f6f83347b16cdfb9153b647c6e5229a822b16724a1bbd9d&scene=38#wechat_redirect) （进阶）

7、[海神平台Crash监控SDK（Android）开发经验总结](https://mp.weixin.qq.com/s/PoWPWy3cXFlG1nohgJTJgw) 

8、[Android Native Crash 收集](https://mp.weixin.qq.com/s?__biz=MzAxNzMxNzk5OQ==&mid=2649485876&idx=1&sn=29ead87814c62a9b3e80cd6ada51e13c&chksm=83f83b34b48fb2225db860a290b4e7ee3b30475d887c2b38e8e29881b297c3cdb5c6d44deaed&scene=38#wechat_redirect) 

9、[理解Android Crash处理流程](http://gityuan.com/2016/06/24/app-crash/)

10、[Android应用ANR分析](https://www.jianshu.com/p/30c1a5ad63a3) 

11、[理解Android ANR的触发原理](http://gityuan.com/2016/07/02/android-anr/)

12、[Input系统—ANR原理分析](http://gityuan.com/2017/01/01/input-anr/)

13、[ANR监测机制](https://www.jianshu.com/p/ad1a84b6ec69) 

14、[理解Android ANR的触发原理](http://gityuan.com/2016/07/02/android-anr/) 

15、[理解Android ANR的信息收集过程](http://gityuan.com/2016/12/02/app-not-response/) 

16、[应用与系统稳定性第一篇---ANR问题分析的一般套路](https://www.jianshu.com/p/18f16aba79dd)

17、[巧妙定位ANR问题](https://www.jianshu.com/p/545e5e7bbf94) 

18、[剖析 SharedPreference apply 引起的 ANR 问题](https://mp.weixin.qq.com/s/IFgXvPdiEYDs5cDriApkxQ) 

19、[Linux错误信号](https://www.mkssoftware.com/docs/man5/siginfo_t.5.asp)


## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd8189413d794abea1af5e1e79fefc4b~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。
