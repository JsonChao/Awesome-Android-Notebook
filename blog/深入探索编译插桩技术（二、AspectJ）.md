---

		title:  深入探索编译插桩技术（二、AspectJ）
		date: 2020/2/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


现如今，编译插桩技术已经深入 `Android` 开发中的各个领域，而 `AOP` 技术正是一种高效实现插桩的模式，它的出现正好给处于黑暗中的我们带来了光明，极大地解决了传统开发过程中的一些痛点，而 `AspectJ` 作为一套基于 `Java` 语言面向切面的扩展设计规范，能够赋予我们新的能力。在这篇文章中我们将来学习如何使用 `AspectJ` 来进行插桩。本篇内容如下所示：


- 1）、**编译插桩技术的分类与应用场景**。
- 2）、**AspectJ 的优势与局限性**。
- 3）、**AspectJ 核心语法简介**。
- 4）、**AspectJX 实战**。
- 5）、**使用 AspectJX 打造自己的性能监控框架**。
- 6）、**总结**。


面向切面的程序设计 `(aspect-oriented programming (AOP))` 吸引了很多开发者的目光， 但是如何在编码中有效地实现这一套设计概念却并不简单，幸运的是，早在 2003 年，一套基于 `Java` 语言面向切面的扩展设计：`AspectJ` 诞生了。

不同与传统的 OOP 编程，`AspectJ (即 AOP)` 的独特之处在于 **发现那些使用传统编程方法无法处理得很好的问题**。
例如一个要在某些应用中实施安全策略的问题。安全性是贯穿于系统所有模块间的问题，而且每一个模块都必须要添加安全性才能保证整个应用的安全性，并且安全性模块自身也需要安全性，很明显这里的 **安全策略的实施问题就是一个横切关注点**，使用传统的编程解决此问题非常的困难而且容易产生差错，这正是 `AOP` 发挥作用的时候了。

传统的面向对象编程中，每个单元就是一个类，而 **类似于安全性这方面的问题，它们通 常不能集中在一个类中处理，因为它们横跨多个类，这就导致了代码无法重用，它们是不可靠和不可继承的，这样的编程方式使得可维护性差而且产生了大量的代码冗余，这是我们所不愿意看到的**。

**而面向切面编程的出现正好给处于黑暗中的我们带来了光明，它针对于这些横切关注点进行处理，就似面向对象编程处理一般的关注点一样**。

在我们继续深入 `AOP` 编程之前，我们有必要先来看看当前编译插桩技术的分类与应用场景。这样能让我们 **从更高的纬度上去理解各个技术点之间的关联与作用**。


# 一、编译插桩技术的分类与应用场景

编译插桩技术具体可以分为两类，如下所示：

- 1）、`APT（Annotation Process Tools）` ：**用于生成 Java 代码**。
- 2）、`AOP（Aspect Oriented Programming）`：**用于操作字节码**。


下面👇，我们分别来详细介绍下它们的作用。

## 1、APT（Annotation Process Tools） 

总所周知，`ButterKnife、Dagger、GreenDao、Protocol Buffers` 这些常用的注解生成框架都会在编译过程中生成代码。而 **这种使用 AndroidAnnotation 结合 APT 技术 来生成代码的时机，是在编译最开始的时候介入的。而 AOP 是在编译完成后生成 dex 文件之前的时候，直接通过修改 .class 文件的方式，来直接添加或者修改代码逻辑的**。

使用 `APT` 技术生成 `Java` 代码的方式具有如下 **两方面** 的优势：

- 1）、**隔离了框架复杂的内部实现，使得开发更加地简单高效**。
- 2）、**大大减少了手工重复的工作量，降低了开发时出错的机率**。


## 2、AOP（Aspect Oriented Programming）

而对于操作字节码的方式来说，一般都在 **代码监控、代码修改、代码分析** 这三个场景有着很广泛的应用。

相对于 `Java` 代码生成的方式，操作字节码的方式有如下 **特点**：

- 1）、**应用场景更广**。
- 2）、**功能更加强大**。
- 3）、**使用复杂度较高**。

此外，我们不仅可以操作 `.class` 文件的 `Java` 字节码，也可以操作 `.dex` 文件的 `Dalvik` 字节码。下面我们就来大致了解下在以上三类场景中编译插桩技术具体是如何应用的。


### 1、代码监控

编译插桩技术除了 **不能够实现耗电监控**，它能够实现各式各样的性能监控，例如：**网络数据监控、耗时方法监控、大图监控、线程监控** 等等。

譬如 **网络数据监控** 的实现，就是在 **网络层通过 hook 网络库方法 和 自动化注入拦截器的形式，实现网络请求的全过程监控，包括获取握手时长，首包时间，DNS 耗时，网络耗时等各个网络阶段的信息**。

实现了对网络请求过程的监控之后，我们便可以 **对整个网络过程的数据表现进行详细地分析，找到网络层面性能的问题点，并做出针对性地优化措施**。例如针对于 `网络错误率偏高` 的问题，我们可以采取以下几方面的措施，如下所示：

- 1）、**使用 HttpDNS**。
- 2）、**将错误数据同步 CDN**。
- 3）、**CDN 调度链路优化**。


### 2、代码修改

用编译插桩技术来实现代码修改的场景非常之多，而使用最为频繁的场景具体可细分为为如下四种：

- 1）、`实现无痕埋点`：**如 [网易HubbleData之Android无埋点实践](https://neyoufan.github.io/2017/07/11/android/%E7%BD%91%E6%98%93HubbleData%E4%B9%8BAndroid%E6%97%A0%E5%9F%8B%E7%82%B9%E5%AE%9E%E8%B7%B5/)、[51 信用卡 Android 自动埋点实践
](https://mp.weixin.qq.com/s/P95ATtgT2pgx4bSLCAzi3Q)**。
- 2）、`统一处理点击抖动`：**编译阶段统一 hook  android.view.View.OnClickListener#onClick() 方法，来实现一个快速点击无效的防抖动效果，这样便能高效、无侵入性地统一解决客户端快速点击多次导致频繁响应的问题**。
- 3）、`第三方 SDK 的容灾处理`：**我们可以在上线前临时修改或者 hook 第三方 SDK 的方法，做到快速容灾上线**。
- 4）、`实现热修复框架`：**我们可以在 Gradle 进行自动化构建的时候，即在 Java 源码编译完成之后，生成 dex 文件之前进行插桩，而插桩的作用是在每个方法执行时先去根据自己方法的签名寻找是否有自己对应的 patch 方法，如果有，执行 patch 方法；如果没有，则执行自己原有的逻辑**。


### 3、代码分析

例如 `Findbugs` 等三方的代码检查工具里面的 **自定义代码检查** 也使用了编译插桩技术，利用它我们可以找出 **不合理的 Hanlder 使用、new Thread 调用、敏感权限调用** 等等一系列编码问题。


# 二、AspectJ 的优势与局限性

最常用的字节码处理框架有 `AspectJ、ASM` 等等，它们的相同之处在于输入输出都是 `Class` 文件。并且，它们都是 **在 Java 文件编译成 .class 文件之后，生成 Dalvik 字节码之前执行**。

而 **AspectJ 作为 Java 中流行的 AOP（aspect-oriented programming） 编程扩展框架，其内部使用的是 [BCEL框架](https://github.com/apache/commons-bcel) 来完成其功能**。下面，我们就来了解下 `AspectJ` 具备哪些优势。


## 1、AspectJ 的优势

它的优势有两点：成熟稳定、使用非常简单。


### 1、成熟稳定

字节码的处理并不简单，特别是 **针对于字节码的格式和各种指令规则**，如果处理出错，就会导致程序编译或者运行过程中出现问题。而 `AspectJ` 作为从 2001 年发展至今的框架，它已经发展地非常成熟，通常不用考虑插入的字节码发生正确性相关的问题。


### 2、使用非常简单

`AspectJ` 的使用非常简单，并且它的功能非常强大，我们完全不需要理解任何 `Java` 字节码相关的知识，就可以在很多情况下对字节码进行操控。例如，它可以在如下五个位置插入自定义的代码：

- 1）、**在方法（包括构造方法）被调用的位置**。
- 2）、**在方法体（包括构造方法）的内部**。
- 3）、**在读写变量的位置**。
- 4）、**在静态代码块内部**。
- 5）、**在异常处理的位置的前后**。

此外，它也可以 **直接将原位置的代码替换为自定义的代码**。


## 2、AspectJ 的缺陷

而 `AspectJ` 的缺点可以归结为如下 **三点**：


### 1、切入点固定

**AspectJ 只能在一些固定的切入点来进行操作**，如果想要进行更细致的操作则很难实现，它无法针对一些特定规则的字节码序列做操作。


### 2、正则表达式的局限性

`AspectJ` 的匹配规则采用了类似正则表达式的规则，比如 **匹配 Activity 生命周期的 onXXX 方法，如果有自定义的其他以 on 开头的方法也会匹配到，这样匹配的正确性就无法满足**。


### 3、性能较低

`AspectJ` 在实现时会包装自己一些特定的类，它并不会直接把 `Trace` 函数直接插入到代码中，而是经过一系列自己的封装。这样不仅生成的字节码比较大，而且对原函数的性能会有不小的影响。**如果想对 App 中所有的函数都进行插桩，性能影响肯定会比较大。如果你只插桩一小部分函数，那么 AspectJ 带来的性能损耗几乎可以忽略不计**。


# 三、AspectJ 核心语法简介

`AspectJ` 其实就是一种 AOP 框架，**AOP 是实现程序功能统一维护的一种技术**。利用 `AOP` 可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合性降低，提高程序的可重用性，同时大大提高了开发效率。因此 `AOP` 的优势可总结为如下 **两点**：

- 1）、**无侵入性**。
- 2）、**修改方便**。


此外，AOP 不同于 OOP 将问题划分到单个模块之中，它把 **涉及到众多模块的同一类问题进行了统一处理**。比如我们可以设计两个切面，一个是用于处理 App 中所有模块的日志输出功能，另外一个则是用于处理 App 中一些特殊函数调用的权限检查。

下面👇，我们就来看看要掌握 AspectJ 的使用，我们需要了解的一些 **核心概念**。


### 1、横切关注点

对哪些方法进行拦截，拦截后怎么处理。

### 2、切面（Aspect）

类是对物体特征的抽象，切面就是对横切关注点的抽象。

### 3、连接点（JoinPoint）

JPoint 是一个程序的关键执行点，也是我们关注的重点。它就是指被拦截到的点（如方法、字段、构造器等等）。

### 4、切入点（PointCut）

对 JoinPoint 进行拦截的定义。PointCut 的目的就是提供一种方法使得开发者能够选择自己感兴趣的 JoinPoint。

### 5、通知（Advice）

切入点仅用于捕捉连接点集合，但是，除了捕捉连接点集合以外什么事情都没有做。事实上实现横切行为我们要使用通知。它 **一般指拦截到 JoinPoint 后要执行的代码，分为 前置、后置、环绕 三种类型**。这里，我们需要**注意 Advice Precedence（优先权） 的情况，比如我们对同一个切面方法同时使用了 @Before 和 @Around 时就会报错，此时会提示需要设置 Advice 的优先级**。

**AspectJ 作为一种基于 Java 语言实现的一套面向切面程序设计规范**。它向 `Java` 中加入了 `连接点(Join Point)` 这个新概念，其实它也只是现存的一个 `Java` 概 念的名称而已。它向 `Java` 语言中加入了少许新结构，譬如 `切入点(pointcut)、通知(Advice)、类型间声明(Inter-type declaration) 和 切面(Aspect)`。**切入点和通知动态地影响程序流程，类型间声明则是 静态的影响程序的类等级结构，而切面则是对所有这些新结构的封装**。

对于 AsepctJ 中的各个核心概念来说，其 <span style="color:rgb(248,57,41);font-weight:bold;">连接点就恰如程序流中适当的一点。而切入点收集特定的连接点集合和在这些点中的值。一个通知则是当一个连接点到达时执行的代码，这些都是 AspectJ 的动态部分。其实连接点就好比是 程序中那一条一条的语句，而切入点就是特定一条语句处设置的一个断点，它收集了断点处程序栈的信息，而通知就是在这个断点前后想要加入的程序代码。</span>

此外，`AspectJ` 中也有许多不同种类的类型间声明，这就允许程序员修改程序的静态结构、名称、类的成员以及类之间的关系。 `AspectJ` 中的切面是横切关注点的模块单元。它们的行为与 `Java` 语言中的类很象，但是切面 还封装了切入点、通知以及类型间声明。

在 `Android` 平台上要使用 `AspectJ` 还是有点麻烦的，这里我们可以直接使用沪江的 [AspectJX](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx) 框架。下面，我们就来使用 `AspectJX` 进行 `AOP` 切面编程。


# 四、AspectJX 实战

首先，为了在 `Android` 使用 `AOP` 埋点需要引入 `AspectJX`，在项目根目录的 `build.gradle` 下加入：

```java
    classpath 'com.hujiang.aspectjx:gradle-android-plugin-aspectjx:2.0.0'
```   

然后，在 `app` 目录下的 `build.gradle` 下加入：

```java
    apply plugin: 'android-aspectjx'
    implement 'org.aspectj:aspectjrt:1.8.+'
```
    
`JoinPoint` 一般定位在如下位置:

- 1）、**函数调用**。
- 2）、**获取、设置变量**。
- 3）、**类初始化**。


**使用 PointCut 对我们指定的连接点进行拦截，通过 Advice，就可以拦截到 JoinPoint 后要执行的代码**。Advice 通常有以下 **三种类型**：

- 1）、**Before：PointCut 之前执行**。
- 2）、**After：PointCut 之后执行**。
- 3）、**Around：PointCut 之前、之后分别执行**。


## 1、最简单的 AspectJ 示例

首先，我们举一个 `小栗子`🌰：

```java
    @Before("execution(* android.app.Activity.on**(..))")
    public void onActivityCalled(JoinPoint joinPoint) throws Throwable {
        Log.d(...)
    }
```

其中，**在 execution 中的是一个匹配规则，第一个 * 代表匹配任意的方法返回值，后面的语法代码匹配所有 Activity 中以 on 开头的方法**。这样，我们就可以 **在 App 中所有 Activity 中以 on 开头的方法中输出一句 log**。

上面的 execution 就是处理 Join Point 的类型，通常有如下两种类型：

- 1）、**call：代表调用方法的位置，插入在函数体外面**。
- 2）、**execution：代表方法执行的位置，插入在函数体内部**。


## 2、统计 Application 中所有方法的耗时

那么，我们如何利用它统计 `Application` 中的所有方法耗时呢？


```java
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
```

需要注意的是，**当 Action 为 Before、After 时，方法入参为 JoinPoint。当 Action 为 Around 时，方法入参为 ProceedingPoint**。

**而 Around 和 Before、After 的最大区别就是 ProceedingPoint 不同于 JoinPoint，其提供了 proceed 方法执行目标方法**。


## 3、对 App 中所有的方法进行 Systrace 函数插桩

在 [《深入探索 Android 启动速度优化》](https://juejin.im/post/5e6f18a951882549422ef333) 一文中我讲到了使用 `Systrace` 对函数进行插桩，从而能够查看应用中方法的耗时与 `CPU` 情况。学习了 `AspectJ` 之后，我们就可以利用它实现对 `App` 中所有的方法进行 `Systrace` 函数插桩了，代码如下所示：


```java
    @Aspect
    public class SystraceTraceAspectj {

        private static final String TAG = "SystraceTraceAspectj";

        @Before("execution(* **(..))")
        public void before(JoinPoint joinPoint) {
            TraceCompat.beginSection(joinPoint.getSignature().toString());
        }
    
        @After("execution(* **(..))")
        public void after() {
            TraceCompat.endSection();
        }
    }
```

了解了 `AspectJX` 的基本使用之后，接下来我们就会使用它和 `AspectJ` 去打造一个简易版的 `APM（性能监控框架）`。


# 五、使用 AspectJ 打造自己的性能监控框架

现在，我们将以奇虎360的 [ArgusAPM](https://github.com/Qihoo360/ArgusAPM) 性能监控框架来全面分析下 AOP 技术在性能监控方面的应用。主要分为如下 **三个部分**：

- 1）、**监控应用冷热启动耗时与生命周期耗时**。
- 2）、**监控 OKHttp3 的每一次网络请求**。
- 3）、**监控 HttpConnection 的每一次网络请求**。


## 1、监控应用冷热启动耗时与生命周期耗时

在 `ArgusAPM` 中，实现了 `Activity` 切面文件 `TraceActivity`， 它被用来监控应用冷热启动耗时与生命周期耗时，`TraceActivity` 的实现代码如下所示：


```java
    @Aspect
    public class TraceActivity {

        // 1、定义一个切入点方法 baseCondition，用于排除 argusapm 中相应的类。
        @Pointcut("!within(com.argusapm.android.aop.*) && !within(com.argusapm.android.core.job.activity.*)")
        public void baseCondition() {
        }

        // 2、定义一个切入点 applicationOnCreate，用于执行 Application 的 onCreate方法。
        @Pointcut("execution(* android.app.Application.onCreate(android.content.Context)) && args(context)")
        public void applicationOnCreate(Context context) {

        }

        // 3、定义一个后置通知 applicationOnCreateAdvice，用于在 application 的 onCreate 方法执行完之后插入 AH.applicationOnCreate(context) 这行代码。
        @After("applicationOnCreate(context)")
        public void applicationOnCreateAdvice(Context context) {
            AH.applicationOnCreate(context);
        }

        // 4、定义一个切入点，用于执行 Application 的 attachBaseContext 方法。
        @Pointcut("execution(* android.app.Application.attachBaseContext(android.content.Context)) && args(context)")
        public void applicationAttachBaseContext(Context context) {
        }

        // 5、定义一个前置通知，用于在 application 的 onAttachBaseContext 方法之前插入 AH.applicationAttachBaseContext(context) 这行代码。
        @Before("applicationAttachBaseContext(context)")
        public void applicationAttachBaseContextAdvice(Context context) {
            AH.applicationAttachBaseContext(context);
        }

        // 6、定义一个切入点，用于执行所有 Activity 中以 on 开头的方法，后面的 ”&& baseCondition()“ 是为了排除 ArgusAPM 中的类。
        @Pointcut("execution(* android.app.Activity.on**(..)) && baseCondition()")
        public void activityOnXXX() {
        }

        // 7、定义一个环绕通知，用于在所有 Activity 的 on 开头的方法中的开始和结束处插入相应的代码。（排除了 ArgusAPM 中的类）
        @Around("activityOnXXX()")
        public Object activityOnXXXAdvice(ProceedingJoinPoint proceedingJoinPoint) {
            Object result = null;
            try {
                Activity activity = (Activity) proceedingJoinPoint.getTarget();
                //        Log.d("AJAOP", "Aop Info" + activity.getClass().getCanonicalName() +
                //                "\r\nkind : " + thisJoinPoint.getKind() +
                //                "\r\nargs : " + thisJoinPoint.getArgs() +
                //                "\r\nClass : " + thisJoinPoint.getClass() +
                //                "\r\nsign : " + thisJoinPoint.getSignature() +
                //                "\r\nsource : " + thisJoinPoint.getSourceLocation() +
                //                "\r\nthis : " + thisJoinPoint.getThis()
                //        );
                long startTime = System.currentTimeMillis();
                result = proceedingJoinPoint.proceed();
                String activityName = activity.getClass().getCanonicalName();

                Signature signature = proceedingJoinPoint.getSignature();
                String sign = "";
                String methodName = "";
                if (signature != null) {
                    sign = signature.toString();
                    methodName = signature.getName();
                }

                if (!TextUtils.isEmpty(activityName) && !TextUtils.isEmpty(sign) && sign.contains(activityName)) {
                    invoke(activity, startTime, methodName, sign);
                }
            } catch (Exception e) {
                e.printStackTrace();
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
            return result;
        }

        public void invoke(Activity activity, long startTime, String methodName, String sign) {
            AH.invoke(activity, startTime, methodName, sign);
        }
    }
```  
    
我们注意到，在注释4、5这两处代码是用于 **在 application 的 onAttachBaseContext 方法之前插入 AH.applicationAttachBaseContext(context) 这行代码**。此外，注释2、3两处的代码是用于 **在 application 的 onCreate 方法执行完之后插入 AH.applicationOnCreate(context) 这行代码**。下面，我们再看看 `AH` 类中这两个方法的实现，代码如下所示：


```java
    public static void applicationAttachBaseContext(Context context) {
        ActivityCore.appAttachTime = System.currentTimeMillis();
        if (Env.DEBUG) {
            LogX.d(Env.TAG, SUB_TAG, "applicationAttachBaseContext time : " + ActivityCore.appAttachTime);
        }
    }

    public static void applicationOnCreate(Context context) {
        if (Env.DEBUG) {
            LogX.d(Env.TAG, SUB_TAG, "applicationOnCreate");
        }

    }
```

可以看到，**在 AH 类的 applicationAttachBaseContext 方法中将启动时间 appAttachTime 记录到了 ActivityCore 实例中**。而 applicationOnCreate 基本上什么也没有实现。

然后，我们再回到切面文件 TraceActivity 中，看到注释6、7处的代码，这里用于 **在所有 Activity 的 on 开头的方法中的开始和结束处插入相应的代码**。需要注意的是，这里 **排除了 ArgusAPM 中的类**。 

下面，我们来分析下 `activityOnXXXAdvice` 方法中的操作。首先，**在目标方法执行前获取了 startTime**。然后，**调用了 proceedingJoinPoint.proceed() 用于执行目标方法**；最后，**调用了 AH 类的 invoke 方法**。我们看看 `invoke` 方法的处理，代码如下所示：


```java
    public static void invoke(Activity activity, long startTime, String lifeCycle, Object... extars) {
        // 1
        boolean isRunning = isActivityTaskRunning();
        if (Env.DEBUG) {
            LogX.d(Env.TAG, SUB_TAG, lifeCycle + " isRunning : " + isRunning);
        }
        if (!isRunning) {
            return;
        }

        // 2
        if (TextUtils.equals(lifeCycle, ActivityInfo.TYPE_STR_ONCREATE)) {
            ActivityCore.onCreateInfo(activity, startTime);
        } else {
            // 3
            int lc = ActivityInfo.ofLifeCycleString(lifeCycle);
            if (lc <= ActivityInfo.TYPE_UNKNOWN || lc > ActivityInfo.TYPE_DESTROY) {
                return;
            }
            ActivityCore.saveActivityInfo(activity, ActivityInfo.HOT_START, System.currentTimeMillis() - startTime, lc);
        }
    }
``` 
    
首先，在注释1处，我们会先去查看当前应用的 `Activity` 耗时统计任务是否打开了。如果打开了，然后就会走到注释2处，这里 **会先判断目标方法名称是否是 “onCreate”，如果是 onCreate 方法，就会执行 ActivityCore 的 onCreateInfo 方法**，代码如下所示：


```java
    // 是否是第一次启动
    public static boolean isFirst = true;
    public static long appAttachTime = 0;
    // 启动类型
    public static int startType;
    
    public static void onCreateInfo(Activity activity, long startTime) {
        // 1   
        startType = isFirst ? ActivityInfo.COLD_START : ActivityInfo.HOT_START;
        // 2
        activity.getWindow().getDecorView().post(new FirstFrameRunnable(activity, startType, startTime));
        //onCreate 时间
        long curTime = System.currentTimeMillis();
        // 3
        saveActivityInfo(activity, startType, curTime - startTime, ActivityInfo.TYPE_CREATE);
    }
```
    
首先，在注释1处，会 **记录此时的启动类型，第一次默认是冷启动**。然后在注释2处，**当第一帧显示时会 post 一个 Runnable**。最后，在注释3处，会 **调用 saveActivityInfo 将目标方法相关的信息保存起来**。这里我们先看看这个 `FirstFrameRunnable` 的 `run` 方法的实现代码，如下所示：


```java
     @Override
        public void run() {
            if (DEBUG) {
                LogX.d(TAG, SUB_TAG, "FirstFrameRunnable time:" + (System.currentTimeMillis() - startTime));
            }
            // 1
            if ((System.currentTimeMillis() - startTime) >= ArgusApmConfigManager.getInstance().getArgusApmConfigData().funcControl.activityFirstMinTime) {
                saveActivityInfo(activity, startType, System.currentTimeMillis() - startTime, ActivityInfo.TYPE_FIRST_FRAME);
            }
            if (DEBUG) {
                LogX.d(TAG, SUB_TAG, "FirstFrameRunnable time:" + String.format("[%s, %s]", ActivityCore.isFirst, ActivityCore.appAttachTime));
            }
            if (ActivityCore.isFirst) {
                ActivityCore.isFirst = false;
                if (ActivityCore.appAttachTime <= 0) {
                    return;
                }
                // 2
                int t = (int) (System.currentTimeMillis() - ActivityCore.appAttachTime);
                AppStartInfo info = new AppStartInfo(t);
                ITask task = Manager.getInstance().getTaskManager().getTask(ApmTask.TASK_APP_START);
                if (task != null) {
                    // 3
                    task.save(info);
                    if (AnalyzeManager.getInstance().isDebugMode()) {
                        // 4
                        AnalyzeManager.getInstance().getParseTask(ApmTask.TASK_APP_START).parse(info);
                    }
                } else {
                    if (DEBUG) {
                        LogX.d(TAG, SUB_TAG, "AppStartInfo task == null");
                    }
                }
            }
        }
    }
```

首先，在注释1处，会计算出当前的 **第一帧的时间**，即 **当前 Activity 的冷启动时间**，将它与 activityFirstMinTime 这个值作比较（activityFirstMinTime 的值默认为300ms），**如果 Activity 的冷启动时间大于300ms的话，就会将冷启动时间调用 saveActivityInfo 方法保存起来**。

然后，在注释2处，我们会 **记录 App 的启动时间** 并在注释3处将它 **保存到 AppStartTask 这个任务实例中**。最后，在注释4处，**如果是 debug 模式，则会调用 AnalyzeManager 这个数据分析管理单例类的 getParseTask 方法获取 AppStartParseTask 这个实例**，关键代码如下所示：


```java
    private Map<String, IParser> mParsers;
    
    private AnalyzeManager() {
        mParsers = new HashMap<String, IParser>(3);
        mParsers.put(ApmTask.TASK_ACTIVITY, new ActivityParseTask());
        mParsers.put(ApmTask.TASK_NET, new NetParseTask());
        mParsers.put(ApmTask.TASK_FPS, new FpsParseTask());
        mParsers.put(ApmTask.TASK_APP_START, new AppStartParseTask());
        mParsers.put(ApmTask.TASK_MEM, new MemoryParseTask());
        this.isUiProcess = Manager.getContext().getPackageName().equals(ProcessUtils.getCurrentProcessName());
    }

    public IParser getParseTask(String name) {
        if (TextUtils.isEmpty(name)) {
            return null;
        }
        return mParsers.get(name);
    }
```

接着，就会调用 `AppStartParseTask` 类的 `parse` 方法，可以看出，它是一个 **专门用于在 Debug 模式下的应用启动时间分析类**。`parse` 方法的代码如下所示：


```java
    /**
     * app启动
     *
     * @param info
     */
    @Override
    public boolean parse(IInfo info) {
        if (info instanceof AppStartInfo) {
            AppStartInfo aInfo = (AppStartInfo) info;
            if (aInfo == null) {
                return false;
            }
            try {
                JSONObject obj = aInfo.toJson();
                obj.put("taskName", ApmTask.TASK_APP_START);
                // 1
                OutputProxy.output("启动时间:" + aInfo.getStartTime(), obj.toString());
            } catch (JSONException e) {
                e.printStackTrace();
            }
            DebugFloatWindowUtls.sendBroadcast(aInfo);
        }
        return true;
    }
```

在注释1处，`parse` 方法中仅仅是继续调用了 `OutputProxy` 的 `output` 方法 **将启动时间和记录启动信息的字符串传入**。我们再看看 `OutputProxy` 的 `output` 方法，如下所示：


```java 
    /**
     * 警报信息输出
     *
     * @param showMsg
     */
    public static void output(String showMsg) {
        if (!AnalyzeManager.getInstance().isDebugMode()) {
            return;
        }
        if (TextUtils.isEmpty(showMsg)) {
            return;
        }
        // 1、存储在本地
        StorageManager.saveToFile(showMsg);
    }
``` 

注释1处，在 `output` 方法中又继续调用了 `StorageManager` 的 `saveToFile` 方法 **将启动信息存储在本地**，`saveToFile` 的实现代码如下所示：


```java
    /**
     * 按行保存到文本文件
     *
     * @param line
     */
    public static void saveToFile(String line) {
        TraceWriter.log(Env.TAG, line);
    }
``` 
    
这里又调用了 `TraceWriter` 的 `log` 方法 **将启动信息按行保存到文本文件中**，关键代码如下所示：


```java
    public static void log(String tagName, String content) {
        log(tagName, content, true);
    }

    private synchronized static void log(String tagName, String content, boolean forceFlush) {
        if (Env.DEBUG) {
            LogX.d(Env.TAG, SUB_TAG, "tagName = " + tagName + " content = " + content);
        }
        if (sWriteThread == null) {
            // 1
            sWriteThread = new WriteFileRun();
            Thread t = new Thread(sWriteThread);
            t.setName("ApmTrace.Thread");
            t.setDaemon(true);
            t.setPriority(Thread.MIN_PRIORITY);
            t.start();

            String initContent = "---- Phone=" + Build.BRAND + "/" + Build.MODEL + "/verName:" + " ----";
            // 2
            sQueuePool.offer(new Object[]{tagName, initContent, Boolean.valueOf(forceFlush)});
            if (Env.DEBUG) {
                LogX.d(Env.TAG, SUB_TAG, "init offer content = " + content);
            }
        }
        if (Env.DEBUG) {
            LogX.d(Env.TAG, SUB_TAG, "offer content = " + content);
        }
        // 3
        sQueuePool.offer(new Object[]{tagName, content, Boolean.valueOf(forceFlush)});

        synchronized (LOCKER_WRITE_THREAD) {
            LOCKER_WRITE_THREAD.notify();
        }
    }
```

在注释1处，**如果 sWriteThread 这个负责写入 log 信息的 Runnable 不存在，就会新建并启动这个写入 log 信息的低优先级守护线程**。

然后，会在注释2处，**调用 sQueuePool 的 offer 方法将相关的信息保存，它的类型为 ConcurrentLinkedQueue，说明它是一个专用于并发环境下的队列**。如果 Runnable 已经存在了的话，就直接会在注释3处将 log 信息入队。最终，**会在 sWriteThread 的 run 方法中调用 sQueuePool 的 poll() 方法将 log 信息拿出并通过 BufferWriter 封装的 FileWriter 将信息保存在本地**。

到此，我们就分析完了 `onCreate` 方法的处理，接着我们再回到 `invoke` 方法的注释3处来分析不是 `onCreate` 方法的情况。**如果方法名不是 onCreate 方法的话，就会调用 ActivityInfo 的 ofLifeCycleString 方法**，我们看看它的实现，如下所示：


```java
    /**
     * 生命周期字符串转换成数值
     *
     * @param lcStr
     * @return
     */
    public static int ofLifeCycleString(String lcStr) {
        int lc = 0;
        if (TextUtils.equals(lcStr, TYPE_STR_FIRSTFRAME)) {
            lc = TYPE_FIRST_FRAME;
        } else if (TextUtils.equals(lcStr, TYPE_STR_ONCREATE)) {
            lc = TYPE_CREATE;
        } else if (TextUtils.equals(lcStr, TYPE_STR_ONSTART)) {
            lc = TYPE_START;
        } else if (TextUtils.equals(lcStr, TYPE_STR_ONRESUME)) {
            lc = TYPE_RESUME;
        } else if (TextUtils.equals(lcStr, TYPE_STR_ONPAUSE)) {
            lc = TYPE_PAUSE;
        } else if (TextUtils.equals(lcStr, TYPE_STR_ONSTOP)) {
            lc = TYPE_STOP;
        } else if (TextUtils.equals(lcStr, TYPE_STR_ONDESTROY)) {
            lc = TYPE_DESTROY;
        }
        return lc;
    }
```
    
可以看到，**ofLifeCycleString 的作用就是将生命周期字符串转换成相应的数值**，下面是它们的定义代码：


```java
    /**
     * Activity 生命周期类型枚举
     */
    public static final int TYPE_UNKNOWN = 0;
    public static final int TYPE_FIRST_FRAME = 1;
    public static final int TYPE_CREATE = 2;
    public static final int TYPE_START = 3;
    public static final int TYPE_RESUME = 4;
    public static final int TYPE_PAUSE = 5;
    public static final int TYPE_STOP = 6;
    public static final int TYPE_DESTROY = 7;
    
    /**
     * Activity 生命周期类型值对应的名称
     */
    public static final String TYPE_STR_FIRSTFRAME = "firstFrame";
    public static final String TYPE_STR_ONCREATE = "onCreate";
    public static final String TYPE_STR_ONSTART = "onStart";
    public static final String TYPE_STR_ONRESUME = "onResume";
    public static final String TYPE_STR_ONPAUSE = "onPause";
    public static final String TYPE_STR_ONSTOP = "onStop";
    public static final String TYPE_STR_ONDESTROY = "onDestroy";
    public static final String TYPE_STR_UNKNOWN = "unKnown";
```

然后，我们再回到 `AH` 类的 `invoke` 方法的注释3处，仅仅当方法名是上述定义的方法，**也就是 Acitivity 的生命周期方法或第一帧的方法时，才会调用 ActivityCore 的 saveActivityInfo 方法**。该方法的实现代码如下所示：


```java
    public static void saveActivityInfo(Activity activity, int startType, long time, int lifeCycle) {
        if (activity == null) {
            if (DEBUG) {
                LogX.d(TAG, SUB_TAG, "saveActivityInfo activity == null");
            }
            return;
        }
        if (time < ArgusApmConfigManager.getInstance().getArgusApmConfigData().funcControl.activityLifecycleMinTime) {
            return;
        }
        String pluginName = ExtraInfoHelper.getPluginName(activity);
        String activityName = activity.getClass().getCanonicalName();
        activityInfo.resetData();
        activityInfo.activityName = activityName;
        activityInfo.startType = startType;
        activityInfo.time = time;
        activityInfo.lifeCycle = lifeCycle;
        activityInfo.pluginName = pluginName;
        activityInfo.pluginVer = ExtraInfoHelper.getPluginVersion(pluginName);
        if (DEBUG) {
            LogX.d(TAG, SUB_TAG, "apmins saveActivityInfo activity:" + activity.getClass().getCanonicalName() + " | lifecycle : " + activityInfo.getLifeCycleString() + " | time : " + time);
        }
        ITask task = Manager.getInstance().getTaskManager().getTask(ApmTask.TASK_ACTIVITY);
        boolean result = false;
        if (task != null) {
            result = task.save(activityInfo);
        } else {
            if (DEBUG) {
                LogX.d(TAG, SUB_TAG, "saveActivityInfo task == null");
            }
        }
        if (DEBUG) {
            LogX.d(TAG, SUB_TAG, "activity info:" + activityInfo.toString());
        }
        if (AnalyzeManager.getInstance().isDebugMode()) {
            AnalyzeManager.getInstance().getActivityTask().parse(activityInfo);
        }
        if (Env.DEBUG) {
            LogX.d(TAG, SUB_TAG, "saveActivityInfo result:" + result);
        }
    }
```  
    
可以看到，这里的逻辑很简单，仅仅是 **将 log 信息保存在 ActivityInfo 这个实例中，并将 ActivityInfo 实例保存在了 ActivityTask 中**，需要注意的是，**在调用 ArgusAPM.init() 这句初始化代码时就已经将 ActivityTask 实例保存在了 taskMap 这个 HashMap 对象中** 了，关键代码如下所示：


```java
    /**
     * 注册 task:每添加一个task都要进行注册，也就是把
     * 相应的 xxxTask 实例放入 taskMap 集合中。
     */
    public void registerTask() {
        if (Env.DEBUG) {
            LogX.d(Env.TAG, "TaskManager", "registerTask " + getClass().getClassLoader());
        }
        if (Build.VERSION.SDK_INT >= 16) {
            taskMap.put(ApmTask.TASK_FPS, new FpsTask());
        }
        taskMap.put(ApmTask.TASK_MEM, new MemoryTask());
        taskMap.put(ApmTask.TASK_ACTIVITY, new ActivityTask());
        taskMap.put(ApmTask.TASK_NET, new NetTask());
        taskMap.put(ApmTask.TASK_APP_START, new AppStartTask());
        taskMap.put(ApmTask.TASK_ANR, new AnrLoopTask(Manager.getContext()));
        taskMap.put(ApmTask.TASK_FILE_INFO, new FileInfoTask());
        taskMap.put(ApmTask.TASK_PROCESS_INFO, new ProcessInfoTask());
        taskMap.put(ApmTask.TASK_BLOCK, new BlockTask());
        taskMap.put(ApmTask.TASK_WATCHDOG, new WatchDogTask());
    }
``` 
    
接着，我们再看看 `ActivityTask` 类的实现，如下所示：


```java
    public class ActivityTask extends BaseTask {

        @Override
        protected IStorage getStorage() {
            return new ActivityStorage();
        }

        @Override
        public String getTaskName() {
            return ApmTask.TASK_ACTIVITY;
        }

        @Override
        public void start() {
            super.start();
            if (Manager.getInstance().getConfig().isEnabled(ApmTask.FLAG_COLLECT_ACTIVITY_INSTRUMENTATION) && !InstrumentationHooker.isHookSucceed()) {//hook失败
                if (DEBUG) {
                    LogX.d(TAG, "ActivityTask", "canWork hook : hook失败");
                }
                mIsCanWork = false;
            }
        }

        @Override
        public boolean isCanWork() {
            return mIsCanWork;
        }
    }
```

可以看到，这里并没有看到 `save` 方法，说明是在基类 `BaseTask` 类中，继续看到 `BaseTask` 类的实现代码：


```java
    /**
    * ArgusAPM任务基类
    *
    * @author ArgusAPM Team
    */
    public abstract class BaseTask implements ITask {

        ...

        @Override
        public boolean save(IInfo info) {
            if (DEBUG) {
                LogX.d(TAG, SUB_TAG, "save task :" + getTaskName());
            }
            // 1
            return info != null && mStorage != null && mStorage.save(info);
        }

        ...
    }
```
    
在注释1处，**继续调用了 mStorage 的 save 方法，它是一个接口 IStorage**，很显然，这里的**实现类是在 ActivityTask 的 getStorage() 方法中返回的 ActivityStorage 实例，它是一个 Activity 存储类，专门负责处理 Activity 的信息**。到此，监控应用冷热启动耗时与生命周期耗时的部分就分析完毕了。
    
下面，我们再看看如何使用 `AspectJ` 监控 `OKHttp3` 的每一次网络请求。


## 2、监控 OKHttp3 的每一次网络请求

首先，我们看到 `OKHttp3` 的切面文件，代码如下所示：


```java
    /**
    * OKHTTP3 切面文件
    *
    * @author ArgusAPM Team
    */
    @Aspect
    public class OkHttp3Aspect {

        // 1、定义一个切入点，用于直接调用 OkHttpClient 的 build 方法。
        @Pointcut("call(public okhttp3.OkHttpClient build())")
        public void build() {

        }

        // 2、使用环绕通知在 build 方法执行前添加一个 NetWokrInterceptor。
        @Around("build()")
        public Object aroundBuild(ProceedingJoinPoint joinPoint) throws Throwable {
            Object target = joinPoint.getTarget();

            if (target instanceof OkHttpClient.Builder && Client.isTaskRunning(ApmTask.TASK_NET)) {
                OkHttpClient.Builder builder = (OkHttpClient.Builder) target;
                builder.addInterceptor(new NetWorkInterceptor());
            }

            return joinPoint.proceed();
        }
    }
```
    
在注释1、2处，**在调用 OkHttpClient 的 build 方法之前添加了一个 NetWokrInterceptor**。我们看看它的实现代码，如下所示：


```java
    @Override
    public Response intercept(Chain chain) throws IOException {
        // 1、获取每一个 OkHttp 请求的开始时间
        long startNs = System.currentTimeMillis();

        mOkHttpData = new OkHttpData();
        mOkHttpData.startTime = startNs;

        if (Env.DEBUG) {
            Log.d(TAG, "okhttp request 开始时间：" + mOkHttpData.startTime);
        }

        Request request = chain.request();
        
        // 2、记录当前请求的请求 url 和请求数据大小
        recordRequest(request);

        Response response;

        try {
            response = chain.proceed(request);
        } catch (IOException e) {
            if (Env.DEBUG) {
                e.printStackTrace();
                Log.e(TAG, "HTTP FAILED: " + e);
            }
            throw e;
        }
        
        // 3、记录这次请求花费的时间
        mOkHttpData.costTime = System.currentTimeMillis() - startNs;

        if (Env.DEBUG) {
            Log.d(TAG, "okhttp chain.proceed 耗时：" + mOkHttpData.costTime);
        }
        
        // 4、记录当前请求返回的响应码和响应数据大小
        recordResponse(response);

        if (Env.DEBUG) {
            Log.d(TAG, "okhttp chain.proceed end.");
        }

        // 5、记录 OkHttp 的请求数据
        DataRecordUtils.recordUrlRequest(mOkHttpData);
        return response;
    }
```
    
首先，在注释1处，**获取了每一个 OkHttp 请求的开始时间**。接着，在注释2处，**通过 recordRequest 方法记录了当前请求的请求 url 和请求数据大小**。然后，注释3处，记录了这次 **请求所花费的时间**。

接下来，在注释4处，**通过 recordResponse 方法记录了当前请求返回的响应码和响应数据大小**。最后，在注释5处，**调用了 DataRecordUtils 的 recordUrlRequest  方法记录了 mOkHttpData 中保存好的数据**。我们继续看到 `recordUrlRequest` 方法，代码如下所示：


```java
    /**
     * recordUrlRequest
     *
     * @param okHttpData
     */
    public static void recordUrlRequest(OkHttpData okHttpData) {
        if (okHttpData == null || TextUtils.isEmpty(okHttpData.url)) {
            return;
        }

        QOKHttp.recordUrlRequest(okHttpData.url, okHttpData.code, okHttpData.requestSize,
                okHttpData.responseSize, okHttpData.startTime, okHttpData.costTime);

        if (Env.DEBUG) {
            Log.d(Env.TAG, "存储okkHttp请求数据，结束。");
        }
    }
```

可以看到，这里**调用了 QOKHttp 的 recordUrlRequest 方法用于记录网络请求信息**。我们再看到 `QOKHttp` 的 `recordUrlRequest` 方法，如下所示：


```java
    /**
     * 记录一次网络请求
     *
     * @param url          请求url
     * @param code         状态码
     * @param requestSize  发送的数据大小
     * @param responseSize 接收的数据大小
     * @param startTime    发起时间
     * @param costTime     耗时
     */
    public static void recordUrlRequest(String url, int code, long requestSize, long responseSize,
                                        long startTime, long costTime) {
        NetInfo netInfo = new NetInfo();
        netInfo.setStartTime(startTime);
        netInfo.setURL(url);
        netInfo.setStatusCode(code);
        netInfo.setSendBytes(requestSize);
        netInfo.setRecordTime(System.currentTimeMillis());
        netInfo.setReceivedBytes(responseSize);
        netInfo.setCostTime(costTime);
        netInfo.end();
    }
```
    
可以看到，这里 **将网络请求信息保存在了 NetInfo 中，并最终调用了 netInfo 的 end 方法**，代码如下所示：


```java
    /**
     * 为什存储的操作要写到这里呢?
     * 历史原因
     */
    public void end() {
        if (DEBUG) {
            LogX.d(TAG, SUB_TAG, "end :");
        }
        this.isWifi = SystemUtils.isWifiConnected();
        this.costTime = System.currentTimeMillis() - startTime;
        if (AnalyzeManager.getInstance().isDebugMode()) {
            AnalyzeManager.getInstance().getNetTask().parse(this);
        }
        ITask task = Manager.getInstance().getTaskManager().getTask(ApmTask.TASK_NET);
        if (task != null) {
            // 1
            task.save(this);
        } else {
            if (DEBUG) {
                LogX.d(TAG, SUB_TAG, "task == null");
            }
        }
    }
```    
    
可以看到，这里 **最终还是调用了 NetTask 实例的 save 方法保存网络请求的信息**。而 **NetTask 肯定是使用了与之对应的 NetStorage 实例将信息保存在了 ContentProvider 中**。至此，`OkHttp3` 这部分的分析就结束了。
    
对于使用 `OkHttp3` 的应用来说，上述的实现可以有效地获取网络请求的信息，但是如果应用没有使用 `OkHttp3` 呢？这个时候，我们就只能去监控 `HttpConnection` 的每一次网络请求。下面，我们就看看如何去实现它。


## 3、监控 HttpConnection 和 HttPClient 的每一次网络请求

在 `ArgusAPM` 中，使用的是 `TraceNetTrafficMonitor` 这个切面类对 `HttpConnection` 的每一次网络请求进行监控。关键代码如下所示：


```java
    @Aspect
    public class TraceNetTrafficMonitor {

        // 1
        @Pointcut("(!within(com.argusapm.android.aop.*) && ((!within(com.argusapm.android.**) && (!within(com.argusapm.android.core.job.net.i.*) && (!within(com.argusapm.android.core.job.net.impl.*) && (!within(com.qihoo360.mobilesafe.mms.transaction.MmsHttpClient) && !target(com.qihoo360.mobilesafe.mms.transaction.MmsHttpClient)))))))")
        public void baseCondition() {
        }

        // 2
        @Pointcut("call(org.apache.http.HttpResponse org.apache.http.client.HttpClient.execute(org.apache.http.client.methods.HttpUriRequest)) && (target(httpClient) && (args(request) && baseCondition()))")
        public void httpClientExecuteOne(HttpClient httpClient, HttpUriRequest request) {
        }

        // 3
        @Around("httpClientExecuteOne(httpClient, request)")
        public HttpResponse httpClientExecuteOneAdvice(HttpClient httpClient, HttpUriRequest request) throws IOException {
            return QHC.execute(httpClient, request);
        }

        // 排查一些处理异常的切面代码

        // 4
        @Pointcut("call(java.net.URLConnection openConnection()) && (target(url) && baseCondition())")
        public void URLOpenConnectionOne(URL url) {
        }

        // 5
        @Around("URLOpenConnectionOne(url)")
        public URLConnection URLOpenConnectionOneAdvice(URL url) throws IOException {
            return QURL.openConnection(url);
        }

        // 排查一些处理异常的切面代码
    
    }
``` 
    
`TraceNetTrafficMonitor` 里面的操作分为 **两类**，**一类是用于切 HttpClient 的 execute 方法**，即注释1、2、3处所示的切面代码；**一类是用于切 HttpConnection 的 openConnection 方法**，对应的切面代码为注释4、5处。我们首先分析 `HttpClient` 的情况，这里最终 **调用了 QHC 的 execute 方法进行处理**，如下所示：


```java
    public static HttpResponse execute(HttpClient client, HttpUriRequest request) throws IOException {
        return isTaskRunning()
                ? AopHttpClient.execute(client, request)
                : client.execute(request);
    }
``` 

这里又 **继续调用了 AopHttpClient 的 execute 方法**，代码如下所示：


```java
    public static HttpResponse execute(HttpClient httpClient, HttpUriRequest request) throws IOException {
        NetInfo data = new NetInfo();
        // 1
        HttpResponse response = httpClient.execute(handleRequest(request, data));
        // 2
        handleResponse(response, data);
        return response;
    }
```   
    
首先，在注释1处，**调用了 handleRequest 处理请求数据**，如下所示：


```java
    private static HttpUriRequest handleRequest(HttpUriRequest request, NetInfo data) {
        data.setURL(request.getURI().toString());
        if (request instanceof HttpEntityEnclosingRequest) {
            HttpEntityEnclosingRequest entityRequest = (HttpEntityEnclosingRequest) request;
            if (entityRequest.getEntity() != null) {
                // 1、将请求实体使用 AopHttpRequestEntity 进行了封装
                entityRequest.setEntity(new AopHttpRequestEntity(entityRequest.getEntity(), data));
            }
            return (HttpUriRequest) entityRequest;
        }
        return request;
    }
 ```   

可以看到，在注释1处，**使用 AopHttpRequestEntity 对请求实体进行了封装**，这里的目的主要是为了 **便于使用封装实体中的 NetInfo 进行数据操作**。接着，在注释2处，将得到的响应信息进行了处理，这里的实现很简单，就是 **使用 NetInfo 这个实体类将响应信息保存在了 ContentProvider 中**。至此，`HttpClient` 的处理部分我们就分析完毕了。

下面，我们接着分析下 `HTTPConnection` 的切面部分代码，如下所示：


```java
    // 4
    @Pointcut("call(java.net.URLConnection openConnection()) && (target(url) && baseCondition())")
    public void URLOpenConnectionOne(URL url) {
    }

    // 5
    @Around("URLOpenConnectionOne(url)")
    public URLConnection URLOpenConnectionOneAdvice(URL url) throws IOException {
        return QURL.openConnection(url);
    }
```  

可以看到，这里是 **调用了 QURL 的 openConnection 方法进行处理**。我们来看看它的实现代码：

 
```java   
    public static URLConnection openConnection(URL url) throws IOException {
        return isNetTaskRunning() ? AopURL.openConnection(url) : url.openConnection();
    }
``` 

这里 **又调用了 AopURL 的 openConnection 方法**，继续
看看它的实现：


```java   
    public static URLConnection openConnection(URL url) throws IOException {
        if (url == null) {
            return null;
        }
        return getAopConnection(url.openConnection());
    }
    
    private static URLConnection getAopConnection(URLConnection con) {
        if (con == null) {
            return null;
        }
        if (Env.DEBUG) {
            LogX.d(TAG, "AopURL", "getAopConnection in AopURL");
        }
        
        // 1
        if ((con instanceof HttpsURLConnection)) {
            return new AopHttpsURLConnection((HttpsURLConnection) con);
        }
        
        // 2
        if ((con instanceof HttpURLConnection)) {
            return new AopHttpURLConnection((HttpURLConnection) con);
        }
        return con;
    }
```
    
最终，在注释1处，**会判断如果是 https 请求，则会使用 AopHttpsURLConnection 封装 con，如果是 http 请求，则使用 AopHttpURLConnection 进行封装**。`AopHttpsURLConnection` 的实现与它类似，仅仅是多加了 `SSL` 证书验证的部分。所以这里我们就直接分析一下 `AopHttpURLConnection` 的实现，这里面的代码非常多，就不贴出来了，但是，它的 **核心的处理** 可以简述为如下 **两点**：

- 1）、**在回调 getHeaderFields()、getInputStream()、getLastModified() 等一系列方法时会调用 inspectAndInstrumentResponse 方法把响应大小和状态码保存在 NetInfo 中**。
- 2）、**在回调 onInputstreamComplete()、onInputstreamError()等方法时，即请求完成或失败时，此时会直接调用 myData 的 end 方法将网络响应信息保存在 ContentProvider 中**。


至此，`ArgusAPM` 的 `AOP` 实现部分就已经全部分析完毕了。


# 六、总结

最后，我们再来回顾一下本篇文章中我们所学到的知识，如下所示：

- 1、**编译插桩技术的分类与应用场景**。
    - 1）、**APT**。
    - 2）、**AOP**。
- 2、**AspectJ 的优势与局限性**。
- 3、**AspectJ 核心语法简介**。
- 4、**AspectJX 实战**。
    - 1）、**最简单的 AspectJ 示例**。
    - 2）、**统计 Application 中所有方法的耗时**。
    - 3）、**对 App 中所有的方法进行 Systrace 函数插桩**。
- 5、**使用 AspectJ 打造自己的性能监控框架**。
    - 1）、**监控应用冷热启动耗时与生命周期耗时**。
    - 2）、**监控 OKHttp3 的每一次网络请求**。
    - 3）、**监控 HttpConnection 和 HttpClient 的每一次网络请求**。


可以看到，`AOP` 技术的确很强大，使用 `AspectJ` 我们能做很多事情，但是，它也有一系列的缺点，比如切入点固定、正则表达式固有的缺陷导致的使用不灵活，此外，它还生成了比较多的包装代码。那么，有没有更好地实现方式，**既能够在使用上更加地灵活，也能够避免生成包装代码，以减少插桩所带来的性能损耗呢**？没错，就是 `ASM`，但是它 **需要通过操作 JVM 字节码的方式来进行代码插桩，入手难度比较大**，所以，下篇文章我们将会先深入学习 `JVM` 字节码的知识，敬请期待~


## 参考链接：
---
1、[极客时间之Android开发高手课 编译插桩的三种方法：AspectJ、ASM、ReDex](https://time.geekbang.org/column/article/82761)

2、[《AspectJ 程序设计指南》PDF](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/ASM3.0%E6%8C%87%E5%8D%97%E7%BF%BB%E8%AF%91.pdf)

3、[The AspectJ 5 Development Kit Developer's Notebook](https://www.eclipse.org/aspectj/doc/next/adk15notebook/index.html)

4、[深入理解Android之AOP](https://blog.csdn.net/Innost/article/details/49387395)

5、[AOP技术在客户端的应用与实践](https://mp.weixin.qq.com/s?__biz=MzUxODg0MzU2OQ==&mid=2247483887&idx=1&sn=d54e3f210a4f31f477dba06c3dcd352e&scene=21#wechat_redirect)

6、[注意Advice Precedence的情况](https://www.eclipse.org/aspectj/doc/released/progguide/semantics-advice.html)

7、[AspectJX](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)

8、[BCEL框架](https://github.com/apache/commons-bcel)

9、[360 的性能监控框架ArgusAPM中AspectJ的使用](https://github.com/Qihoo360/ArgusAPM/tree/master/argus-apm/argus-apm-aop/src/main/java/com/argusapm/android/aop)

10、[利用AspectJ实现插桩的例子](https://github.com/AndroidAdvanceWithGeektime/Chapter27)


# Contanct Me

##  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

##  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

![](https://user-gold-cdn.xitu.io/2020/3/30/171292d8d046013d?w=1080&h=2047&f=jpeg&s=76889)
        

##  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


## About me

- ### Email: [chao.qu521@gmail.com]()
- ### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- ### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。