---

		title:  深度探索 Gradle 自动化构建技术（五、Gradle 插件架构实现原理剖析）
		date: 2020/2/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


目前，Gradle 自动化技术越来越重要，也有许多同学已经能够制作出自己的 Gradle 插件，**但是一直有一些 “梗” 遗留在我们脑海中，无时无刻不提醒着我们，你真的掌握了吗**？例如，**“梗1”：Gradle 插件的整体实现架构**？我：...，**“梗2”：Android Gradle 插件更新历史有哪些重要优化或者改进**？我：...,   **“梗3”：Gradle 构建的核心流程是怎样的**？我：...，**“梗4”：Gradle 中依赖实现的原理**？我：..., **“梗5”：AppPlugin 构建流程**？我：..., **“梗6”：assembleDebug 打包流程**？我：...., **“梗7”:一些很重要 Task 实现原理能否说说**？我：... 。是否有很多点并没有真正地去了解过呢？


# 思维导图

![](https://user-gold-cdn.xitu.io/2020/4/27/171bb38ec49777f4?w=3052&h=2788&f=png&s=1054813)


# 目录

- 一、[Gradle 插件实现架构概述](https://juejin.im/post/5ea7792e5188256da0323444#heading-4)
- 二、[了解 Android Gradle Plugin 的更新历史](https://juejin.im/post/5ea7792e5188256da0323444#heading-5)
    - 1、Android Gradle Plugin V3.5.0（2019 年 8 月）
    - 2、Android Gradle Plugin V3.4.0（2019 年 4 月）
    - 3、Android Gradle Plugin V3.3.0（2019 年 1 月）
    - 4、Android Gradle Plugin V3.2.0（2018 年 9 月）
    - 5、Android Gradle Plugin V3.1.0（2018 年 3 月）
    - 6、Android Gradle Plugin V3.0.0（2017 年 10 月）
    - 7、Android Gradle Plugin V2.3.0（2017 年 2 月）
- 三、[Gradle 构建核心流程解析](https://juejin.im/post/5ea7792e5188256da0323444#heading-28)
    - 1、LoadSettings
    - 2、Configure
    - 3、TaskGraph
    - 4、RunTasks
    - 5、Finished
- 四、[关于 Gradle 中依赖实现的原理](https://juejin.im/post/5ea7792e5188256da0323444#heading-43)
    - 1、通过 MethodMissing 机制，间接地调用 DefaultDependencyHandler 的 add 方法去添加依赖。
    - 2、不同的依赖声明，其实是由不同的转换器进行转换的。
    - 3、Project 依赖的本质是 Artifacts 依赖，也即 产物依赖。
    - 4、什么是 Gradle 中的 Configuration？
    - 5、Task 是如何发布自己 Artifacts 的？
- 五、[AppPlugin 构建流程](https://juejin.im/post/5ea68729f265da47e1596037#heading-2)
    - 1、准备工作
    - 2、configureProject 配置项目
    - 3、configureExtension 配置 Extension
    - 4、TaskManager#createTasksBeforeEvaluate 创建不依赖 flavor 的 task
    - 5、BasePlugin#createAndroidTasks 创建构建 task
- 六、[assembleDebug 打包流程浅析](https://juejin.im/post/5ea68729f265da47e1596037#heading-8)
    - 1、Android 打包流程回顾
    - 2、assmableDebug 打包流程浅析
- 七、[重要 Task 实现源码分析](https://juejin.im/post/5ea68729f265da47e1596037#heading-13)
    - 1、资源处理相关 Task
    - 2、将 Class 文件打包成 Dex 文件的过程
- 八、[总结](https://juejin.im/post/5ea68729f265da47e1596037#heading-44)
    - 最后的最后


# 一、Gradle 插件实现架构概述

Android Gradle plugin 团队在 Android Gradle V3.2.0 之前一直是都是用 Java 编写的 Gradle 插件，在 V3.2.0 便采用了 Kotlin 进行大面积的重写。尽管 Groovy 语法简洁，且其闭包的写法非常灵活，但是 Android Studio 对 Groovy 的支持非常不友好，因此，**目前写自定义的 Gradle 插件时我们还是尽量使用 Kotlin，这样能尽量避免编写插件时提示不够造成的坑**。

下面，我们就来看看 **Gradle 插件的整体实现架构**，如下图所示：


![](https://user-gold-cdn.xitu.io/2020/4/27/171ba83821461652?w=434&h=580&f=png&s=38095)


**在最下层的是底层 Gradle 框架，它主要提供一些基础的服务，如 task 的依赖，有向无环图的构建等等**。

上面的则是 Google 编译工具团队的 `Android Gradle plugin` 框架，它主要是 **在 Gradle 框架的基础上，创建了很多与 Android 项目打包有关的 task 及 artifacts（每一个 task 执行完成之后通常都会输出产物）**。

最上面的则是开发者自定义的 Plugin，关于自定义 Plugin 通常有两种使用套路，如下所示：

- 1）、**在 Android Gradle plugin 提供的 task 的基础上，插入一些自定义的 task**。
- 2）、**增加 Transform 进行编译时代码注入**。


# 二、了解 Android Gradle Plugin 的更新历史

下表列出了各个 Android Gradle 插件版本所需的 Gradle 版本。**我们应该使用 Android Gradle Plugin 与 Gradle 这两者的最新版本以获得最佳的性能**。

插件版本 |	所需的 Gradle 版本
---|---|
1.0.0 - 1.1.3 |	2.2.1 - 2.3
1.2.0 - 1.3.1 |	2.2.1 - 2.9
1.5.0 |	2.2.1 - 2.13
2.0.0 - 2.1.2 |	2.10 - 2.13
2.1.3 - 2.2.3 |	2.14.1+
2.3.0+ |	3.3+
3.0.0+ |	4.1+
3.1.0+ |	4.4+
3.2.0 - 3.2.1 |	4.6+
3.3.0 - 3.3.2 |	4.10.1+
3.4.0 - 3.4.1 |	5.1.1+
3.5.0+	| 5.4.1-5.6.4


目前最新的 Android Gradle Plugin 版本为 V3.6.2，Gradle 版本为 V5.6.4。下面，我们了解下 **Android Gradle Plugin 更新历史中比较重要的更新变化**。


## 1、Android Gradle Plugin V3.5.0（2019 年 8 月）

本次更新的重中之重是 **提高项目的构建速度**。


## 2、Android Gradle Plugin V3.4.0（2019 年 4 月）

**如果使用的是 Gradle 5.0 及更高版本，默认的 Gradle 守护进程内存堆大小会从 1 GB 降到 512 MB。这可能会导致构建性能降低。如果要替换此默认设置，请在项目的 gradle.properties 文件中指定 Gradle 守护进程堆大小**。

### 1）、新的 Lint 检查依赖项配置

**增加了新的依赖项配置 lintPublish，并更改了原有 lintChecks 的行为**，它们的作用分别如下所示：

- `lintChecks`：**仅用于在本地构建项目时运行的 Lint 检查**。
- `lintPublish`：**在已发布的 AAR 中启用 Lint 检查，这样使用此 AAR 的项目也会应用那些 Lint 检查**。


其示例代码如下所示：


```groovy
dependencies {
      // Executes lint checks from the ':lint' project at build time.
      lintChecks project(':lint')
      // Packages lint checks from the ':lintpublish' in the published AAR.
      lintPublish project(':lintpublish')
}
```


### 2）、Android 免安装应用功能插件弃用警告

在之前的版本可以使用 `com.android.feature` 插件构建免安装应用，现在建议使用动态功能插件，这样便可以通过单个 Android App Bundle 发布安装版应用和免安装应用。


### 3）、R8 默认处于启用状态

**R8 将 desugar（脱糖:将 .class 字节码转换为 .dex 字节码的过程）、压缩、混淆、优化和 dex 处理整合到了一个步骤中，从而显著提升了构建性能。R8 是在 Android Gradle Plugin V3.2.0 中引入的，对于使用插件 V3.4.0 及更高版本的应用和 Android 库项目来说，R8 已经默认处于启用状态**。

#### R8 引入之前的编译流程


![](https://user-gold-cdn.xitu.io/2020/4/27/171ba84fcf8c61f8?w=1648&h=398&f=png&s=88962)


#### R8 引入之后的编译流程


![](https://user-gold-cdn.xitu.io/2020/4/27/171ba90ec74d628d?w=1754&h=430&f=png&s=78093)


可以看到，**R8 组合了 Proguard、D8 的功能**。如果遇到因 R8 导致的编译失败的问题，可以配置以下代码停用 R8：


```groovy
# Disables R8 for Android Library modules only.
android.enableR8.libraries = false
# Disables R8 for all modules.
android.enableR8 = false
```


### 3）、弃用 ndkCompile

此时使用 ndkBuild 编译原生库会收到构建错误。我们应该 **使用 CMake 或 ndk-build 将 C 和 C++ 代码添加到项目中**。


## 3、Android Gradle Plugin V3.3.0（2019 年 1 月）

### 1）、为库项目更快地生成 R 类

**在老版本中，Android Gradle 插件会为项目的每个依赖项生成一个 R.java 文件，然后将这些 R 类和应用的其他类一起编译。现在，插件会直接生成包含应用的已编译 R 类的 JAR，而不会先编译中间的 R.java 类。这不仅可以显著提升包含多个库子项目和依赖项的项目的编译性能，还可以加快在 Android Studio 中索引文件的速度**。


## 4、Android Gradle Plugin V3.2.0（2018 年 9 月）

### 新的代码压缩器 R8

R8 是一种执行代码压缩和混淆的新工具，替代了 ProGuard。我们只需将以下代码添加到项目的 gradle.properties 文件中，即可开始使用 R8 的预览版本：


```groovy
android.enableR8 = true
```

### 使用 D8 进行 desugar 的功能现已默认处于启用状态



## 5、Android Gradle Plugin V3.1.0（2018 年 3 月）

### 新的 DEX 编译器 (D8)

**默认情况下，Android Studio 此时会使用名为 D8 的新 DEX 编译器。DEX 编译是指针对 ART （对于较早版本的 Android，则针对 Dalvik）将 .class 字节码转换为 .dex 字节码的过程。与之前的编译器（DX）相比，D8 的编译速度更快，输出的 DEX 文件更小，同时却能保持相同甚至更出色的应用运行时性能**。

如果在使用 D8 的过程中出现了问题，可以在 gradle.properties 配置以下代码暂时停用 D8 并使用 DX：


```groovy
android.enableD8=false
```


## 6、Android Gradle Plugin V3.0.0（2017 年 10 月）

### 1）、通过精细控制的任务图提升了多模块项目的并行性。

**更改依赖项时，Gradle 通过不重新编译那些无法被访问的依赖项 API 模块来加快编译速度**。此时可以利用 Gradle 的新依赖项配置（implementation、api、compileOnly 和 runtimeOnly）限制哪些依赖项会将其 API 泄露给其他模块。


### 2）、借助每个类的 dex 处理，可加快增量编译速度。

**每个类现在都会编译成单独的 DEX 文件，并且只会对修改过的类重新进行 dex 处理**。


### 3）、启用 Gradle 编译缓存优化某些任务来使用缓存的输出

启用 Gradle 编译缓存能够优化某些任务来使用缓存的输出，从而加快编译速度。


### 4）、AAPT2 默认已启用并改进了增量资源处理

如果在使用 AAPT2 时遇到了问题，我们可以停用 AAPT2，在 gradle.properties 文件中设置如下代码：


```groovy
android.enableAapt2=false
```


然后，**通过在命令行中运行 ./gradlew --stop 来重启 Gradle 守护进程**。


## 7、Android Gradle Plugin V2.3.0（2017 年 2 月）

### 1）、增加编译缓存

存储编译项目时 Android 插件生成的特定输出。使用缓存时，编译插件的速度会明显加快，因为编译系统在进行后续编译时可以直接重用这些缓存文件，而不必重新创建。此外，我们也可以 **使用 cleanBuildCache Task 去清除编译缓存**。


更老版本的的更新细节请查阅 [gradle-plugin](https://developer.android.google.cn/studio/releases/gradle-plugin#updating-plugin)。


# 三、Gradle 构建核心流程解析

当我们输入 ./gradlew/gradle ... 命令之后，一个 Gradle 构建就开始了。它包含如下 `三个步骤`：

- 1）、首先，**初始化 Gradle 构建框架自身**。
- 2）、然后，**把命令行参数包装好送给 DefaultGradleLauncher**。
- 3）、最后，**触发 DefaultGradleLauncher 中 Gradle 构建的生命周期，并开始执行标准的构建流程**。


在开始深入源码之前，我们可以先自顶向下了解下 Gradle 构建的核心流程图，以便对 Gradle 的构建流程建立一个整体的感知。

![](https://user-gold-cdn.xitu.io/2020/4/27/171ba96b6c46b423?w=1640&h=5186&f=png&s=1756030)


**当我们执行一个 gralde 命令时，便会调用 gradle/wrapper/gradle-wrapper.jar 里面 org.gradle.wrapper.GradleWrapperMain 类的 main 方法，它就是 gradle 的一个入口方法**。该方法的核心源码如下所示：


```java
public static void main(String[] args) throws Exception {
            
        ...
        
        // 1、索引到 gradle-wrapper.properties 文件中配置的 gradle zip 地址，并由此包装成一个 WrapperExecutor 实例。
        WrapperExecutor wrapperExecutor = WrapperExecutor.forWrapperPropertiesFile(propertiesFile);
        // 2、使用 wrapperExecutor 实例的 execute 方法执行 gradle 命令。
        wrapperExecutor.execute(args, new Install(logger, new Download(logger, "gradlew", "0"), new PathAssembler(gradleUserHome)), new BootstrapMainStarter());
}
```


然后，我们继续看看  wrapperExecutor 的 execute 方法，源码如下所示：


```java
public void execute(String[] args, Install install, BootstrapMainStarter bootstrapMainStarter) throws Exception {
        // 1、下载 gradle wrapper 需要的依赖与源码。
        File gradleHome = install.createDist(this.config);
        // 2、从这里开始执行 gradle 的构建流程。
        bootstrapMainStarter.start(args, gradleHome);
    }
```


首先，在注释1处，**会下载 gradle wrapper 需要的依赖与源码**。接着，在注释2处，**便会调用 bootstrapMainStarter 的 start 方法从这里开始执行 gradle 的构建流程。其内部最终会依次调用 DefaultGradleLauncher 的 getLoadedSettings、getConfiguredBuild、executeTasks 与 finishBuild 方法**，而它们对应的状态都定义在 DefaultGradleLauncher 中的 Stage 枚举类中，如下所示：


```java
private static enum Stage {
        LoadSettings,
        Configure,
        TaskGraph,
        RunTasks {
            String getDisplayName() {
                return "Build";
            }
        },
        Finished;

        private Stage() {
        }

        String getDisplayName() {
            return this.name();
        }
}
```


下面，我们就对这五个流程来进行详细地分析。


## 1、LoadSettings

当调用 getLoadedSettings 方法时便开始了加载 Setting.gradle 的流程。其源码如下所示：


```java
public SettingsInternal getLoadedSettings() {
        this.doBuildStages(DefaultGradleLauncher.Stage.LoadSettings);
        return this.gradle.getSettings();
}
```


这里又继续调用了 doBuildStages 方法进行处理，内部实现如下所示：


```java
private void doBuildStages(DefaultGradleLauncher.Stage upTo) {
        Preconditions.checkArgument(upTo != DefaultGradleLauncher.Stage.Finished, "Stage.Finished is not supported by doBuildStages.");

        try {
            // 当 Stage 是 RunTask 的时候执行。
            if (upTo == DefaultGradleLauncher.Stage.RunTasks && this.instantExecution.canExecuteInstantaneously()) {
                this.doInstantExecution();
            } else {
               // 当 Stage 不是 RunTask 的时候执行。 this.doClassicBuildStages(upTo);
            }
        } catch (Throwable var3) {
            this.finishBuild(upTo.getDisplayName(), var3);
        }

}
```


继续调用 doClassicBuildStages 方法，源码如下所示：


```java
private void doClassicBuildStages(DefaultGradleLauncher.Stage upTo) {
        // 1、当 Stage 为 LoadSettings 时执行 prepareSettings 方法去配置并生成 Setting 实例。
        this.prepareSettings();
        if (upTo != DefaultGradleLauncher.Stage.LoadSettings) {
            // 2、当 Stage 为 Configure 时执行 prepareProjects 方法去配置工程。
            this.prepareProjects();
            if (upTo != DefaultGradleLauncher.Stage.Configure) {
                // 3、当 Stage 为 TaskGraph 时执行 prepareTaskExecution 方法去构建 TaskGraph。
                this.prepareTaskExecution();
                if (upTo != DefaultGradleLauncher.Stage.TaskGraph) {
                   
                   // 4、当 Stage 为 RunTasks 时执行 saveTaskGraph 方法 与 runWork 方法保存 TaskGraph 并执行相应的 Tasks。 this.instantExecution.saveTaskGraph();
                   this.runWork();
                }
            }
        }
}
```


可以看到，`doClassicBuildStages` 方法是个很重要的方法，它对所有的 Stage 任务进行了分发，这里小结一下：

- 1）、**当 Stage 为 LoadSettings 时执行 prepareSettings 方法去配置并生成 Setting 实例**。
- 2）、**当 Stage 为 Configure 时执行 prepareProjects 方法去配置工程**。
- 3）、**当 Stage 为 TaskGraph 时执行 prepareTaskExecution 方法去构建 TaskGraph**。
- 4）、**当 Stage 为 RunTasks 时执行 saveTaskGraph 方法 与 runWork 方法保存 TaskGraph 并执行相应的 Tasks**。


然后，我们接着继续看看 prepareSettings 方法，其源码如下所示：


```java
private void prepareSettings() {
        if (this.stage == null) {
            // 1、回调 BuildListener.buildStarted() 回调接口。
            this.buildListener.buildStarted(this.gradle);
            // 2、调用 settingPreparer 接口的实现类 DefaultSettingsPreparer 的 prepareSettings 方法。
            this.settingsPreparer.prepareSettings(this.gradle);
            this.stage = DefaultGradleLauncher.Stage.LoadSettings;
        }
}
```


在 `prepareSettings` 方法做了两件事：


- 1）、**回调 BuildListener.buildStarted 接口**。
- 2）、**调用 settingPreparer 接口的实现类 DefaultSettingsPreparer 的 prepareSettings 方法**。


我们继续看到 DefaultSettingsPreparer 的 prepareSettings 方法，如下所示：


```java
public void prepareSettings(GradleInternal gradle) {
        // 1、执行 init.gradle，它会在每个项目 build 之前被调用，用于做一些初始化的操作。
        this.initScriptHandler.executeScripts(gradle);
        SettingsLoader settingsLoader = gradle.getParent() != null ? this.settingsLoaderFactory.forNestedBuild() : this.settingsLoaderFactory.forTopLevelBuild();
        // 2、调用 SettingLoader 接口的实现类 DefaultSettingsLoader 的 findAndLoadSettings 找到 Settings.gradle 文件的位置。
        settingsLoader.findAndLoadSettings(gradle);
}
```


在 `prepareSettings` 方法中做了两项处理：

- 1）、**执行 init.gradle，它会在每个项目 build 之前被调用，用于做一些初始化的操作**。
- 2）、**调用 SettingLoader 接口的实现类 DefaultSettingsLoader 的 findAndLoadSettings 找到 Settings.gradle 文件的位置**。


DefaultSettingLoader 的 findAndLoadSettings 方法关联的实现代码非常多，限于篇幅，我这里直接点出 **findAndLoadSettings 方法中的主要处理流程**：


- 1）、首先，**查找 settings.gradle 位置**。
- 2）、然后，**编译 buildSrc（Android 默认的 Plugin 目录）文件夹下的内容**。
- 3）、接着，**解析 gradle.properites 文件：这里会读取 gradle.properties 文件里的配置信息与命令行传入的配置属性并存储**。
- 4）、然后，**解析 settings.gradle 文件：这里最后会调用 BuildOperationScriptPlugin.apply 去执行 settings.gradle 文件**。
- 5）、最后，**根据 Settings.gradle 文件中获得的信息去创建 project 以及 subproject 实例**。


## 2、Configure

当执行完 LoadSetting 阶段之后，就会执行 Configure 阶段，而配置阶段所作的事情就是 **把 gradle 脚本编译成 class 文件并执行**。由前可知，此时会执行 prepareProjects 方法，如下所示：


```java
private void prepareProjects() {
        if (this.stage == DefaultGradleLauncher.Stage.LoadSettings) {
            // 1、调用 ProjectsPreparer 接口的实现类  DefaultProjectsPreparer 的 prepareProjects 方法。
            this.projectsPreparer.prepareProjects(this.gradle);
            this.stage = DefaultGradleLauncher.Stage.Configure;
        }
}
```


这里会继续调用 ProjectsPreparer 接口的实现类  DefaultProjectsPreparer 的 prepareProjects 方法。其源码如下所示：


```java
public void prepareProjects(GradleInternal gradle) {
        
        ...

        // 1、如果在 gradle.properties 文件中指定了参数 configure-on-demand，则只会配置主项目以及执行 task 所需要的项目。
        if (gradle.getStartParameter().isConfigureOnDemand()) {
            this.projectConfigurer.configure(gradle.getRootProject());
        } else {
            // 2、如果没有指定在 gradle.properties 文件中指定参数 configure-on-demand，则会调用 ProjectConfigurer 接口的实现类 TaskPathProjectEvaluator 的 configureHierarchy 方法去配置所有项目。
            this.projectConfigurer.configureHierarchy(gradle.getRootProject());
            (new ProjectsEvaluatedNotifier(this.buildOperationExecutor)).notify(gradle);
        }

        this.modelConfigurationListener.onConfigure(gradle);
}
```


在注释1处，**如果在 gradle.properties 文件中指定了参数 configure-on-demand，则只会配置主项目以及执行 task 所需要的项目。我们这里只看默认没有指定的情况**。否则，在注释2处，则会调用 ProjectConfigurer 接口的实现类 TaskPathProjectEvaluator 的 configureHierarchy 方法去配置所有项目。

我们接着继续看到 configureHierarchy 方法，如下所示：


```java
public void configureHierarchy(ProjectInternal project) {
        this.configure(project);
        Iterator var2 = project.getSubprojects().iterator();

        while(var2.hasNext()) {
            Project sub = (Project)var2.next();
            this.configure((ProjectInternal)sub);
        }
}
```


可以看到在 configureHierarchy 方法中使用了 Iterator 遍历并配置了所有 Project。**而 configure 方法最终会调用到 EvaluateProject 类的 run 方法**，如下所示：


```java
public void run(final BuildOperationContext context) {
            this.project.getMutationState().withMutableState(new Runnable() {
                public void run() {
                    try {
                        
                        EvaluateProject.this.state.toBeforeEvaluate();
                       
                        // 1、 回调 ProjectEvaluationListener 的 beforeEvaluate 接口。 
                        LifecycleProjectEvaluator.this.buildOperationExecutor.run(new LifecycleProjectEvaluator.NotifyBeforeEvaluate(EvaluateProject.this.project, EvaluateProject.this.state));
                       
                        if (!EvaluateProject.this.state.hasFailure()) {
                            EvaluateProject.this.state.toEvaluate();

                            try {
                               // 2、在 evaluate 方法中会设置默认的 init、wrapper task 和 默认插件，然后便会编译、执行 build.gradle 脚本
                               LifecycleProjectEvaluator.this.delegate.evaluate(EvaluateProject.this.project, EvaluateProject.this.state);
                            } catch (Exception var10) {
                                LifecycleProjectEvaluator.addConfigurationFailure(EvaluateProject.this.project, EvaluateProject.this.state, var10, context);
                            } finally {
                                EvaluateProject.this.state.toAfterEvaluate();
                                // 3、回调  ProjectEvaluationListener.afterEvaluate 接口。     
                                LifecycleProjectEvaluator.this.buildOperationExecutor.run(new LifecycleProjectEvaluator.NotifyAfterEvaluate(EvaluateProject.this.project, EvaluateProject.this.state));
                            }
                        }

                        if (EvaluateProject.this.state.hasFailure()) {
                            EvaluateProject.this.state.rethrowFailure();
                        } else {
                            context.setResult(ConfigureProjectBuildOperationType.RESULT);
                        }
                    } finally {
                        EvaluateProject.this.state.configured();
                    }

                }
            });
}
```


在 EvaluateProject 的 run 方法中有如下 **三个重要的处理**：


- 1）、**回调 ProjectEvaluationListener 的 beforeEvaluate 接口**。 
- 2）、**在 evaluate 方法中会设置默认的 init、wrapper task 和 默认插件，然后便会编译并执行 build.gradle 脚本**。
- 3）、**回调  ProjectEvaluationListener.afterEvaluate 接口**。


## 3、TaskGraph

执行完 初始化阶段 与 配置阶段 之后，就会 **调用到 DefaultGradleLauncher 的 prepareTaskExecution 方法去创建一个由 Tasks 组成的一个有向无环图**。该方法如下所示：


```java
private void prepareTaskExecution() {
        if (this.stage == DefaultGradleLauncher.Stage.Configure) {
            // 1、调用 TaskExecutionPreparer 接口的实现类 BuildOperatingFiringTaskExecutionPreparer 的 prepareForTaskExecution 方法。
            this.taskExecutionPreparer.prepareForTaskExecution(this.gradle);
            this.stage = DefaultGradleLauncher.Stage.TaskGraph;
        }
}
```


这里继续调用了 TaskExecutionPreparer 接口的实现类 BuildOperatingFiringTaskExecutionPreparer 的 prepareForTaskExecution 方法，如下所示：


```java
 public void prepareForTaskExecution(GradleInternal gradle) {
        this.buildOperationExecutor.run(new BuildOperatingFiringTaskExecutionPreparer.CalculateTaskGraph(gradle));
}
```


可以看到，这里使用 **buildOperationExecutor 实例执行了 CalculateTaskGraph 这个构建操作**，我们看到它的 run 方法，如下所示：


```java
public void run(BuildOperationContext buildOperationContext) {
            // 1、填充任务图
            final TaskExecutionGraphInternal taskGraph = this.populateTaskGraph();
            buildOperationContext.setResult(new Result() {
                 getRequestedTaskPaths() {
                    return this.toTaskPaths(taskGraph.getRequestedTasks());
                }

                
                public List<String> getExcludedTaskPaths() {
                    return this.toTaskPaths(taskGraph.getFilteredTasks());
                }

                private List<String> toTaskPaths(Set<Task> tasks) {
                    return ImmutableSortedSet.copyOf(Collections2.transform(tasks, new Function<Task, String>() {
                        public String apply(Task task) {
                            return task.getPath();
                        }
                    })).asList();
                }
            });
}
```


在注释1处，**直接调用了 populateTaskGraph 填充了 Tasks 有向无环图**。源码如下所示：


```java
TaskExecutionGraphInternal populateTaskGraph() {           
            // 1、这里又调用了 TaskExecutionPreparer 接口的另一个实现类 DefaultTaskExecutionPreparer 的 prepareForTaskExecution 方法。
            BuildOperatingFiringTaskExecutionPreparer.this.delegate.prepareForTaskExecution(this.gradle);
            return this.gradle.getTaskGraph();
}
```


可以看到，在注释1处，又调用了 TaskExecutionPreparer 接口的另一个实现类 DefaultTaskExecutionPreparer 的 prepareForTaskExecution 方法。该方法如下所示：


```java
public void prepareForTaskExecution(GradleInternal gradle) {
        // 1
        this.buildConfigurationActionExecuter.select(gradle);
        TaskExecutionGraphInternal taskGraph = gradle.getTaskGraph();
        // 2、根据 Tasks 与 Tasks 间的依赖信息填充 taskGraph 实例。
        taskGraph.populate();
        this.includedBuildControllers.populateTaskGraphs();
        if (gradle.getStartParameter().isConfigureOnDemand()) {
            (new ProjectsEvaluatedNotifier(this.buildOperationExecutor)).notify(gradle);
        }
}
```


在注释1处，调用了 **buildConfigurationActionExecuter 接口的 select 方法，这里采用了 策略 + 接口隔离 设计模式，后续会依次调用 ExcludedTaskFilteringBuildConfigurationAction 的 configure 方法、 DefaultTasksBuildExecutionAction 的 configure 方法、TaskNameResolvingBuildConfigurationAction  的 configure 方法**。

下面，我们依次来分析下其中的处理。


### 1）、ExcludedTaskFilteringBuildConfigurationAction#configure：处理需要排除的 task


```java
public void configure(BuildExecutionContext context) {
        // 1、获取被排除的 Tasks 名称，并依次把它们放入 filters 中。
        GradleInternal gradle = context.getGradle();
        Set<String> excludedTaskNames = gradle.getStartParameter().getExcludedTaskNames();
        if (!excludedTaskNames.isEmpty()) {
            Set<Spec<Task>> filters = new HashSet();
            Iterator var5 = excludedTaskNames.iterator();

            while(var5.hasNext()) {
                String taskName = (String)var5.next();
                filters.add(this.taskSelector.getFilter(taskName));
            }

            // 2、给 TaskGraph 实例添加要 filter 的 Tasks
            gradle.getTaskGraph().useFilter(Specs.intersect(filters));
        }

        context.proceed();
}
```


在注释1处，先获取了被排除的 Tasks 名称，并依次把它们放入 filters 中，接着在注释2处给 TaskGraph 实例添加了要 filter 的 Tasks。

可以看到，**这里给 TaskGraph 设置了 filter 去处理需要排除的 task，以便在后面计算依赖的时候排除相应的 task**。


### 2）、DefaultTasksBuildExecutionAction#configure：添加默认的 task


```java
public void configure(BuildExecutionContext context) {
        StartParameter startParameter = context.getGradle().getStartParameter();
        Iterator var3 = startParameter.getTaskRequests().iterator();

        TaskExecutionRequest request;
        // 1、如果指定了要执行的 Task，则什么也不做。
        do {
            if (!var3.hasNext()) {
                ProjectInternal project = context.getGradle().getDefaultProject();
                this.projectConfigurer.configure(project);
                List<String> defaultTasks = project.getDefaultTasks();
                // 2、如果没有指定 DefaultTasks，则输出 gradle help 信息。
                if (defaultTasks.size() == 0) {
                    defaultTasks = Collections.singletonList("help");
                    LOGGER.info("No tasks specified. Using default task {}", GUtil.toString(defaultTasks));
                } else {
                    LOGGER.info("No tasks specified. Using project default tasks {}", GUtil.toString(defaultTasks));
                }

                // 3、否则，添加默认的 Task。
                startParameter.setTaskNames(defaultTasks);
                context.proceed();
                return;
            }

            request = (TaskExecutionRequest)var3.next();
        } while(request.getArgs().isEmpty());

        context.proceed();
}
```


可以看到，DefaultTasksBuildExecutionAction 类的 configure 方法的处理分为如下三步：


- 1、**如果指定了要执行的 Task，则什么也不做**。
- 2、**如果没有指定 defaultTasks，则输出 gradle help 信息**。
- 3、**否则，添加默认的 defaultTasks**。


### 3）、TaskNameResolvingBuildConfigurationAction#configure：计算 task 依赖图

```java
public void configure(BuildExecutionContext context) {
        GradleInternal gradle = context.getGradle();
        TaskExecutionGraphInternal taskGraph = gradle.getTaskGraph();
        List<TaskExecutionRequest> taskParameters = gradle.getStartParameter().getTaskRequests();
        Iterator var5 = taskParameters.iterator();

        while(var5.hasNext()) {
            TaskExecutionRequest taskParameter = (TaskExecutionRequest)var5.next();
            // 1、解析 Tasks。
            List<TaskSelection> taskSelections = this.commandLineTaskParser.parseTasks(taskParameter);
            Iterator var8 = taskSelections.iterator();

            // 2、遍历并添加所有已选择的 Tasks 至 taskGraph 实例之中。
            while(var8.hasNext()) {
                TaskSelection taskSelection = (TaskSelection)var8.next();
                LOGGER.info("Selected primary task '{}' from project {}", taskSelection.getTaskName(), taskSelection.getProjectPath());
                taskGraph.addEntryTasks(taskSelection.getTasks());
            }
        }

        context.proceed();
}
```


这里主要做了 `两个处理`：

- 1）、`解析 Tasks`：**分析命令行中的 Task 的参数前面是否指定了 Project，例如 ':project:assembleRelease'，如果没有指定则选中 Project 下所有符合该 taskName 的 Task**。
- 2）、`遍历并添加所有已选择的 Tasks 至 taskGraph 实例之中`： **在内部会处理 dependson finalizedby mustrunafter shouldrunafter 等 Tasks 间的依赖关系，并会把信息保存在 TaskInfo 之中**。


### 4）、填充 task 依赖图


最后，我们再回到 DefaultTaskExecutionPreparer 的 prepareForTaskExecution 方法，在注释2处，我们就可以 **调用 taskGraph 的 populate 方法去根据这些 Tasks 与 Tasks 之间的依赖信息去填充 taskGraph 实例了**。


## 4、RunTasks

填充完 taskGraph 之后，我们就可以开始来执行这些 Tasks 了，我们看到 DefaultGradleLauncher 实例的 executeTasks 方法，如下所示：


```java
public GradleInternal executeTasks() {
        this.doBuildStages(DefaultGradleLauncher.Stage.RunTasks);
        return this.gradle;
}
```


在 doBuildStages 方法中又会调用 doClassicBuildStages 方法，源码如下所示：


```java
 private void doClassicBuildStages(DefaultGradleLauncher.Stage upTo) {
        this.prepareSettings();
        if (upTo != DefaultGradleLauncher.Stage.LoadSettings) {
            this.prepareProjects();
            if (upTo != DefaultGradleLauncher.Stage.Configure) {
                this.prepareTaskExecution();
                if (upTo != DefaultGradleLauncher.Stage.TaskGraph) {
                    this.instantExecution.saveTaskGraph();
                    // 1
                    this.runWork();
                }
            }
        }
}
```


在注释1处，继续调用了 runWork 方法，如下所示：


```java
private void runWork() {
        if (this.stage != DefaultGradleLauncher.Stage.TaskGraph) {
            throw new IllegalStateException("Cannot execute tasks: current stage = " + this.stage);
        } else {
            List<Throwable> taskFailures = new ArrayList();
            // 1
            this.buildExecuter.execute(this.gradle, taskFailures);
            if (!taskFailures.isEmpty()) {
                throw new MultipleBuildFailures(taskFailures);
            } else {
                this.stage = DefaultGradleLauncher.Stage.RunTasks;
            }
        }
}
```


这里又继续 **调用了 buildExecuter 接口的实现类  DefaultBuildWorkExecutor 的 execute 方法**，其实现如下所示：


```java
public void execute(GradleInternal gradle, Collection<? super Throwable> failures) {
        this.execute(gradle, 0, failures);
}

private void execute(final GradleInternal gradle, final int index, final Collection<? super Throwable> taskFailures) {
        if (index < this.executionActions.size()) {
            // 1
            ((BuildExecutionAction)this.executionActions.get(index)).execute(new BuildExecutionContext() {
                public GradleInternal getGradle() {
                    return gradle;
                }

                public void proceed() {
                  // 2、执行完 executionActions 列表中的第一项后，便开始执行下一项。
                    DefaultBuildWorkExecutor.this.execute(gradle, index + 1, taskFailures);
                }
            }, taskFailures);
        }
}   
```


### 1）、执行 DryRunBuildExecutionAction：处理 DryRun

可以看到，这里又调用了 BuildExecutionAction 接口的实现类 DryRunBuildExecutionAction 的 execute 方法，如下所示：


```java
 public void execute(BuildExecutionContext context, Collection<? super Throwable> taskFailures) {
        GradleInternal gradle = context.getGradle();
        // 1、如果命令行里包含 --dry-run 参数，则会跳过该 Task 的执行，并输出 Task 的名称与执行的先后关系。
        if (gradle.getStartParameter().isDryRun()) {
            Iterator var4 = gradle.getTaskGraph().getAllTasks().iterator();

            while(var4.hasNext()) {
                Task task = (Task)var4.next();
                this.textOutputFactory.create(DryRunBuildExecutionAction.class).append(((TaskInternal)task).getIdentityPath().getPath()).append(" ").style(Style.ProgressStatus).append("SKIPPED").println();
            }
        } else {
            context.proceed();
        }
}
```


在注释1处，**如果命令行里包含了 --dry-run 参数，则会跳过该 Task 的执行，并输出 Task 的名称与执行的先后关系**。


### 2）、执行：SelectedTaskExecutionAction：使用线程执行被选择的 Tasks

执行完 DryRunBuildExecutionAction 后，我们再回到 DefaultBuildWorkExecutor 类的 execute 方法，在注释2处，**会执行 executionActions 列表中的下一项，即第二项：SelectedTaskExecutionAction**，我们看看它的 execute 方法，如下所示：


```java
public void execute(BuildExecutionContext context, Collection<? super Throwable> taskFailures) {
        
        ...

        taskGraph.addTaskExecutionGraphListener(new SelectedTaskExecutionAction.BindAllReferencesOfProjectsToExecuteListener());
        
        // 1、
        taskGraph.execute(taskFailures);
}
```
 

可以看到，这里直接调用了 TaskExecutionGraphInternal 接口的实现类 DefaultTaskExecutionGraph 的 execute 方法去执行 Tasks，其关键源码如下所示：


```java
public void execute(Collection<? super Throwable> failures) {
        ProjectExecutionServiceRegistry projectExecutionServices = new ProjectExecutionServiceRegistry();

        try {
            // 使用线程池执行 Task
            this.executeWithServices(projectExecutionServices, failures);
        } finally {
            projectExecutionServices.close();
        }
}
```

这里又继续调用了 DefaultTaskExecutionGraph 的 executeWithServices 方法使用线程池并行执行 Task，其核心源码如下所示：


```java
private void executeWithServices(ProjectExecutionServiceRegistry projectExecutionServices, Collection<? super Throwable> failures) {
        
        ...
        
        this.ensurePopulated();
        
        ...

        try {
            // 1
            this.planExecutor.process(this.executionPlan, failures, new DefaultTaskExecutionGraph.BuildOperationAwareExecutionAction(this.buildOperationExecutor.getCurrentOperation(), new DefaultTaskExecutionGraph.InvokeNodeExecutorsAction(this.nodeExecutors, projectExecutionServices)));
            LOGGER.debug("Timing: Executing the DAG took " + clock.getElapsed());
        } finally {
            ...
        }
}
```


最终，这里是 **调用了 PlanExecutor 接口的实现类 DefaultPlanExecutor 的 process 方法**，如下所示：


```java
public void process(ExecutionPlan executionPlan, Collection<? super Throwable> failures, Action<Node> nodeExecutor) {
        ManagedExecutor executor = this.executorFactory.create("Execution worker for '" + executionPlan.getDisplayName() + "'");

        try {
            WorkerLease parentWorkerLease = this.workerLeaseService.getCurrentWorkerLease();
            // 1、开始使用线程池异步执行 tasks
            this.startAdditionalWorkers(executionPlan, nodeExecutor, executor, parentWorkerLease);
            (new DefaultPlanExecutor.ExecutorWorker(executionPlan, nodeExecutor, parentWorkerLease, this.cancellationToken, this.coordinationService)).run();
            this.awaitCompletion(executionPlan, failures);
        } finally {
            executor.stop();
        }
}
```

在注释1处，使用了线程池去异步执行 tasks，如下所示：


```java
private void startAdditionalWorkers(ExecutionPlan executionPlan, Action<? super Node> nodeExecutor, Executor executor, WorkerLease parentWorkerLease) {
        LOGGER.debug("Using {} parallel executor threads", this.executorCount);

        for(int i = 1; i < this.executorCount; ++i) {
            executor.execute(new DefaultPlanExecutor.ExecutorWorker(executionPlan, nodeExecutor, parentWorkerLease, this.cancellationToken, this.coordinationService));
        }
}
```


需要了解的是，**用于执行 Task 的线程默认是 8 个线程。这里使用线程池执行了 DefaultPlanExecutor 的 ExecutorWorker，在它的 run 方法中最终会调用到 DefaultBuildOperationExecutor 的 run 方法去执行 Task。但在 Task 执行前还需要做一些预处理**。


### 3）、Task 执行前的一些预处理

在 DefaultBuildOperationExecutor 的 run 方法中只做了两件事，这里我们只需大致了解下即可，如下所示：


- 1、首先，**回调 TaskExecutionListener#beforeExecute**。
- 2、然后，**链式执行一系列对 Task 的通用处理**，其具体的处理如下所示：
    - 1）、`CatchExceptionTaskExecuter#execute`：**增加 try catch，避免执行过程中发生异常**。
    - 2）、`ExecuteAtMostOnceTaskExecuter#execute`：**判断 task 是否执行过**。
    - 3）、`SkipOnlyIfTaskExecuter#execute`：**判断 task 的 onlyif 条件是否满足执行**。
    - 4）、`SkipTaskWithNoActionsExecuter#execute`：**跳过没有 action 的 task，如果没有 action 说明 task 不需要执行**。
    - 5）、`ResolveTaskArtifactStateTaskExecuter#execute`：**设置 artifact 的状态**。
    - 6）、`SkipEmptySourceFilesTaskExecuter#execut`：**跳过设置了 source file 且 source file 为空的 task，如果 source file 为空则说明 task 没有需要处理的资源**。
    - 7）、`ValidatingTaskExecuter#execute`：**确认 task 是否可以执行**。
    - 8）、`ResolveTaskOutputCachingStateExecuter#execute`：**处理 task 输出缓存**。
    - 9）、`SkipUpToDateTaskExecuter#execute`：**跳过 update-to-date 的 task**。
    - 10）、`ExecuteActionsTaskExecuter#execute`：**用来真正执行 Task 的 executer**。


可以看到，**在真正执行 Task 前需要经历一些通用的 task 预处理，最后才会调用 ExecuteActionsTaskExecuter 的 execute 方法去真正执行 Task**。


### 4）、真正执行 Task

```java
public TaskExecuterResult execute(final TaskInternal task, final TaskStateInternal state, final TaskExecutionContext context) {
        final ExecuteActionsTaskExecuter.TaskExecution work = new ExecuteActionsTaskExecuter.TaskExecution(task, context, this.executionHistoryStore, this.fingerprinterRegistry, this.classLoaderHierarchyHasher);
        // 使用 workExecutor 对象去真正执行 Task。
        final CachingResult result = (CachingResult)this.workExecutor.execute(new AfterPreviousExecutionContext() {
            public UnitOfWork getWork() {
                return work;
            }

            public Optional<String> getRebuildReason() {
                return context.getTaskExecutionMode().getRebuildReason();
            }

            public Optional<AfterPreviousExecutionState> getAfterPreviousExecutionState() {
                return Optional.ofNullable(context.getAfterPreviousExecution());
            }
        });
        ...
}
```


可以看到，这里使用了 workExecutor 对象去真正执行 Task，在执行时便会回调 ExecuteActionsTaskExecuter.TaskExecution 内部类的 execute 方法，其实现源码如下所示：


```java
 public WorkResult execute(@Nullable InputChangesInternal inputChanges) {
            this.task.getState().setExecuting(true);

            WorkResult var2;
            try {
                ExecuteActionsTaskExecuter.LOGGER.debug("Executing actions for {}.", this.task);
               
                // 1、回调 TaskActionListener 的 beforeActions 接口。
                ExecuteActionsTaskExecuter.this.actionListener.beforeActions(this.task);
               
                // 2、内部会遍历执行所有的 Action。 
                ExecuteActionsTaskExecuter.this.executeActions(this.task, inputChanges);
                var2 = this.task.getState().getDidWork() ? WorkResult.DID_WORK : WorkResult.DID_NO_WORK;
            } finally {
                this.task.getState().setExecuting(false);
               
               // 3、回调 TaskActionListener.afterActions。 
               ExecuteActionsTaskExecuter.this.actionListener.afterActions(this.task);
            }

            return var2;
}
```


**在 ExecuteActionsTaskExecuter.TaskExecution 内部类的 execute 方法中做了三件事**，如下所示：


- 1）、**回调 TaskActionListener 的 beforeActions**。
- 2）、**内部会遍历执行 Task 所有的 Action。 => 执行 Task 就是执行其中包含的所有 Action，这便是 Task 的本质**。
- 3）、**回调 TaskActionListener.afterActions**。


**当执行完 Task 的所有 Action 之后，便会最终在 DefaultBuildOperationExecutor 的 run 方法中回调 TaskExecutionListener.afterExecute 来标识 Task 最终执行完成**。


## 5、Finished

在 Finished 阶段仅仅只干了一件比较重要的事情，就是 **回调 buildListener 的 buildFinished 接口**，如下所示：


```java
private void finishBuild(String action, @Nullable Throwable stageFailure) {
        if (this.stage != DefaultGradleLauncher.Stage.Finished) {
            ...

            try {
               
                 this.buildListener.buildFinished(buildResult);
            } catch (Throwable var7) {
                failures.add(var7);
            }

            this.stage = DefaultGradleLauncher.Stage.Finished;
            
            ...
        }
}
```


## 6、小结

从对 Gradle 执行 Task 的分析中可以看到 Task 的本质，其实就是一系列的 Actions。深入了解了 Gradle 的构建流程之后，我们再重新回归头来看看开始的那一张 Gradle 的构建流程图，如下所示：


![](https://user-gold-cdn.xitu.io/2020/4/27/171ba96b6c46b423?w=1640&h=5186&f=png&s=1756030)


此外，zhangyi54 做的 [Gradle原理动画讲解](https://www.bilibili.com/video/av55941638/) 将枯燥的原理流程视觉化了，值得推荐。需要注意区分的是，动画讲解的 Gradle 版本比较老，但是主要的原理还是一致的，可以放心观看。


> 注意：build.gradle 脚本的 buildscript 会在脚本的其它内容之前执行。


# 四、关于 Gradle 中依赖实现的原理

## 1、通过 MethodMissing 机制，间接地调用 DefaultDependencyHandler 的 add 方法去添加依赖

我们都知道，DependencyHandler 是用来进行依赖声明的一个类，但是在 DependencyHandler 并没有发现 implementation(), api(), compile() 这些方法，这是怎么回事呢？

其实这是 **通过 MethodMissing 机制，间接地调用 DependencyHandler 的实现 DefaultDependencyHandler 的 add() 方法将依赖添加进去的**。

### 那么，methodMissing 又是什么？

它是 **Groovy 语言的一个重要特性， 这个特性允许在运行时 catch 对于未定义方法的调用**。

而 **Gradle 对这个特性进行了封装，一个类要想使用这个特性，只要实现 MixIn 接口即可**。代码如下所示：


```java
public interface MethodMixIn {
    MethodAccess getAdditionalMethods();
}
```


## 2、不同的依赖声明，其实是由不同的转换器进行转换的

例如如下两个依赖：

- 1）、`DependencyStringNotationConverter`： **将类似于 'androidx.appcompat:appcompat:1.1.0' 的常规依赖声明转换为依赖**。
- 2）、`DependencyProjectNotationConverter`： **将类似于 'project(":mylib")' 的依赖声明转换为依赖**。


此外，**除了 project 依赖，其它的依赖最终都会转换为 SelfResolvingDependency, 而这个依赖可以实现自我解析**。

## 3、Project 依赖的本质是 Artifacts 依赖，也即 产物依赖。


## 4、什么是 Gradle 中的 Configuration？

**implementation 与 api。但是，承担 moudle 或者 aar 依赖仅仅是 Configuration 表面的任务，依赖的实质就是获取被依赖项构建过程中产生的 Artifacts**。

**而对于大部分的 Task 来说，执行完之后都会有产物输出。而在各个 moudle 之间 Artifacts 的产生与消费则构成了一个完整的生产者-消费者模型**。


## 5、Task 是如何发布自己 Artifacts 的？

**每一个 task 都会调用 VariantScopeImpl 中的 publishIntermediateArtifact 方法将自己的产物进行发布，最后会调用到  DefaultVariant 的 artifact 方法对产物进行发布**。


# 五、AppPlugin 构建流程

为了能够查看 Android Gradle Plugin 与 Gradle 的源码，我们需要在项目中添加 android gradle plugin 依赖，如下所示：

```groovy
compile 'com.android.tools.build:gradle:3.0.1'
```


众所周知，我们能够将一个 moudle 构建为一个 Android 项目是由于在 build.gradle 中配置了如下的插件应用代码：

```groovy
apply plugin: 'com.android.application'
```

**当执行到 apply plugin: 'com.android.application' 这行配置时，也就开始了 AppPlugin 的构建流程**，下面我们就来分析下 AppPlugin 的构建流程。

'com.android.application' 对应的插件 properties 为 'com.android.internal.application'，内部标明的插件实现类如下所示：


```groovy
implementation-class=com.android.build.gradle.internal.plugins.AppPlugin
```


AppPlugin 中的关键实现代码如下所示：


```java
/** Gradle plugin class for 'application' projects, applied on the base application module */
public class AppPlugin extends AbstractAppPlugin {
    ...

    // 应用指定的 plugin，这里是一个空实现
    @Override
    protected void pluginSpecificApply(@NonNull Project project) {
    }

    ...

    // 获取一个扩展类：应用 application plugin 都会提供一个与之对应的 android extension
    @Override
    @NonNull
    protected Class<? extends AppExtension> getExtensionClass() {
        return BaseAppModuleExtension.class;
    }

    ...
}
```


那 apply 方法是在什么地方被调用的呢？

首先，我们梳理下 AppPlugin 的继承关系，如下所示：

```
AppPlugin => AbstractAppPlugin => BasePlugin
```


**而 apply 方法就在 BasePlugin 类中，BasePlugin 是一个应用于所有 Android Plugin 的基类，在 apply 方法中会预先进行一些准备工作**。


## 1、准备工作

当编译器执行到 apply plugin 这行 groovy 代码时，gradle 便会最终回调 BasePlugin 基类 的 apply 方法，如下所示：

```java
@Override
public final void apply(@NonNull Project project) {
    CrashReporting.runAction(
            () -> {
                basePluginApply(project);
                pluginSpecificApply(project);
            });
}
```


在 apply 方法中调用了 basePluginApply 方法，其源码如下所示：


```java
private void basePluginApply(@NonNull Project project) {
       
        ...
        
        // 1、DependencyResolutionChecks 会检查并确保在配置阶段不去解析依赖。
        DependencyResolutionChecks.registerDependencyCheck(project, projectOptions);

        // 2、应用一个 AndroidBasePlugin，目的是为了让其他插件作者区分当前应用的是一个 Android 插件。
        project.getPluginManager().apply(AndroidBasePlugin.class);

        // 3、检查 project 路径是否有错误，发生错误则抛出 StopExecutionException 异常。
        checkPathForErrors();
        
        // 4、检查子 moudle 的结构：目前版本会检查 2 个模块有没有相同的标识（组+名称），如果有则抛出 StopExecutionException 异常。（组件化在不同的 moudle 中需要给资源加 prefix 前缀）
        checkModulesForErrors();

        // 5、插件初始化，必须立即执行。此外，需要注意，Gradle Deamon 永远不会同时执行两个构建流程。
        PluginInitializer.initialize(project);
        
        // 6、初始化用于记录构建过程中配置信息的工厂实例 ProcessProfileWriterFactory
        RecordingBuildListener buildListener = ProfilerInitializer.init(project, projectOptions);
        
        // 7、给 project 设置 android plugin version、插件类型、插件生成器、project  选项
        ProcessProfileWriter.getProject(project.getPath())
                .setAndroidPluginVersion(Version.ANDROID_GRADLE_PLUGIN_VERSION)
                .setAndroidPlugin(getAnalyticsPluginType())
                .setPluginGeneration(GradleBuildProject.PluginGeneration.FIRST)
                .setOptions(AnalyticsUtil.toProto(projectOptions));

        // 配置工程
        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_CONFIGURE,
                project.getPath(),
                null,
                this::configureProject);

        // 配置 Extension
        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,
                project.getPath(),
                null,
                this::configureExtension);

        // 创建 Tasks
        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_TASKS_CREATION,
                project.getPath(),
                null,
                this::createTasks);
    }
```


可以看到，前 4 个步骤都是一些检查操作，而后 3 个步骤则是对插件进行初始化与配置。我们 `梳理下 准备工程 中的任务`，如下所示：


- 1、`插件检查操作`
    - 1）、**使用 DependencyResolutionChecks 类去检查并确保在配置阶段不去解析依赖**。
    - 2）、**应用一个 AndroidBasePlugin，目的是为了让其他插件作者区分当前应用的是一个 Android 插件**。
    - 3）、**检查 project 路径是否有错误，发生错误则抛出 StopExecutionException 异常**。
    - 4）、**检查子 moudle 的结构，目前版本会检查 2 个模块有没有相同的标识（组 + 名称），如果有则抛出 StopExecutionException 异常。（联想到组件化在不同的 moudle 中需要给资源加 prefix 前缀）**
- 2、`对插件进行初始化与配置相关信息`
    - 1）、**立即执行插件初始化**。
    - 2）、**初始化 用于记录构建过程中配置信息的工厂实例 ProcessProfileWriterFactory**。
    - 3）、**给 project 设置 android plugin version、插件类型、插件生成器、project 选项**。
    


## 2、configureProject 配置项目

Plugin 的准备工程完成之后，就会执行 BasePlugin 中的 configureProject 方法进行项目的配置了，其源码如下所示：


```java
private void configureProject() {
        
        ...
        
        // 1、创建 DataBindingBuilder 实例。
        dataBindingBuilder = new DataBindingBuilder();
        dataBindingBuilder.setPrintMachineReadableOutput(
                SyncOptions.getErrorFormatMode(projectOptions) == ErrorFormatMode.MACHINE_PARSABLE);

        // 2、强制使用不低于当前所支持的最小插件版本，否则会抛出异常。
        GradlePluginUtils.enforceMinimumVersionsOfPlugins(project, syncIssueHandler);

        // 3、应用 Java Plugin。
        project.getPlugins().apply(JavaBasePlugin.class);

        // 4、如果启动了 构建缓存 选项，则会创建 buildCache 实例以便后面能重用缓存。
        @Nullable
        FileCache buildCache = BuildCacheUtils.createBuildCacheIfEnabled(project, projectOptions);

        // 5、这个回调将会在整个 project 执行完成之后执行（注意不是在当前 moudle 执行完成之后执行），因为每一个 project 都会调用此回调, 所以它可能会执行多次。
        // 在整个 project 构建完成之后，会进行资源回收、缓存清除并关闭在此过程中所有启动的线程池组件。
        gradle.addBuildListener(
                new BuildAdapter() {
                    @Override
                    public void buildFinished(@NonNull BuildResult buildResult) {
                        // Do not run buildFinished for included project in composite build.
                        if (buildResult.getGradle().getParent() != null) {
                            return;
                        }
                        ModelBuilder.clearCaches();
                        Workers.INSTANCE.shutdown();
                        sdkComponents.unload();
                        SdkLocator.resetCache();
                        ConstraintHandler.clearCache();
                        CachedAnnotationProcessorDetector.clearCache();
                        threadRecorder.record(
                                ExecutionType.BASE_PLUGIN_BUILD_FINISHED,
                                project.getPath(),
                                null,
                                () -> {
                                    if (!projectOptions.get(
                                            BooleanOption.KEEP_SERVICES_BETWEEN_BUILDS)) {
                                        WorkerActionServiceRegistry.INSTANCE
                                                .shutdownAllRegisteredServices(
                                                        ForkJoinPool.commonPool());
                                    }
                                    Main.clearInternTables();
                                });
                        DeprecationReporterImpl.Companion.clean();
                    }
                });
        
        ...
}
```


最后，我们梳理下 configureProject 中所执行的 **五项主要任务**，如下所示：


- 1）、**创建 DataBindingBuilder 实例**。
- 2）、**强制使用不低于当前所支持的最小插件版本，否则会抛出异常**。
- 3）、**应用 Java Plugin**。
- 4）、**如果启动了 构建缓存 选项，则会创建 buildCache 实例以便后续能重用缓存**。
- 5）、**这个回调将会在整个 project 执行完成之后执行（注意不是在当前 moudle 执行完成之后执行），因为每一个 project 都会调用此回调, 所以它可能会执行多次。最后，在整个 project 构建完成之后，会进行资源回收、缓存清除并关闭在此过程中所有启动的线程池组件**。


## 3、configureExtension 配置 Extension

然后，我们看到 BasePlugin 的 configureExtension 方法，其核心源码如下所示：


```java
private void configureExtension() {
        
        // 1、创建盛放 buildType、productFlavor、signingConfig 的容器实例。
        ...

        final NamedDomainObjectContainer<BaseVariantOutput> buildOutputs =
                project.container(BaseVariantOutput.class);

        // 2、创建名为 buildOutputs 的扩展属性配置。
        project.getExtensions().add("buildOutputs", buildOutputs);

        ...

        // 3、创建 android DSL 闭包。
        extension =
                createExtension(
                        project,
                        projectOptions,
                        globalScope,
                        buildTypeContainer,
                        productFlavorContainer,
                        signingConfigContainer,
                        buildOutputs,
                        sourceSetManager,
                        extraModelInfo);

        // 4、给全局域设置创建好的 android DSL 闭包。
        globalScope.setExtension(extension);

        // 5、创建一个 ApplicationVariantFactory 实例，以用于生产 APKs。
        variantFactory = createVariantFactory(globalScope);

        // 6、创建一个 ApplicationTaskManager 实例，负责为 Android 应用工程去创建 Tasks。
        taskManager =
                createTaskManager(
                        globalScope,
                        project,
                        projectOptions,
                        dataBindingBuilder,
                        extension,
                        variantFactory,
                        registry,
                        threadRecorder);

        // 7、创建一个 VariantManager 实例，用于去创建与管理 Variant。
        variantManager =
                new VariantManager(
                        globalScope,
                        project,
                        projectOptions,
                        extension,
                        variantFactory,
                        taskManager,
                        sourceSetManager,
                        threadRecorder);
                        
         // 8、将 whenObjectAdded callbacks 映射到 singingConfig 容器之中。
        signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);

        // 9、如果不是 DynamicFeature（负责添加一个可选的 APK 模块），则会初始化一个 debug signingConfig DSL 对象并设置给默认的 buildType DSL。
        buildTypeContainer.whenObjectAdded(
                buildType -> {
                    if (!this.getClass().isAssignableFrom(DynamicFeaturePlugin.class)) {
                        SigningConfig signingConfig =
                                signingConfigContainer.findByName(BuilderConstants.DEBUG);
                        buildType.init(signingConfig);
                    } else {
                        // initialize it without the signingConfig for dynamic-features.
                        buildType.init();
                    }
                    variantManager.addBuildType(buildType);
                });

        // 10、将 whenObjectAdded callbacks 映射到 productFlavor 容器之中。
        productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);

        // 11、将 whenObjectRemoved 映射在容器之中，当 whenObjectRemoved 回调执行时，会抛出 UnsupportedAction 异常。
        signingConfigContainer.whenObjectRemoved(
                new UnsupportedAction("Removing signingConfigs is not supported."));
        buildTypeContainer.whenObjectRemoved(
                new UnsupportedAction("Removing build types is not supported."));
        productFlavorContainer.whenObjectRemoved(
                new UnsupportedAction("Removing product flavors is not supported."));

        // 12、按顺序依次创建 signingConfig debug、buildType debug、buildType release 类型的 DSL。
        variantFactory.createDefaultComponents(
                buildTypeContainer, productFlavorContainer, signingConfigContainer);
}
```


最后，我们梳理下 configureExtension 中的任务，如下所示：

- 1）、创建盛放 buildType、productFlavor、signingConfig 的容器实例。
- 2）、创建名为 buildOutputs 的扩展属性配置。
- 3）、创建 android DSL 闭包。
- 4）、给全局域设置创建好的 android DSL 闭包。
- 5）、创建一个 ApplicationVariantFactory 实例，以用于生产 APKs。
- 6）、创建一个 ApplicationTaskManager 实例，负责为 Android 应用工程去创建 Tasks。
- 7）、创建一个 VariantManager 实例，用于去创建与管理 Variant。
- 8）、将 whenObjectAdded callbacks 映射到 singingConfig 容器之中。
- 9）、将 whenObjectAdded callbacks 映射到 buildType 容器之中。如果不是 DynamicFeature（负责添加一个可选的 APK 模块），则会初始化一个 debug signingConfig DSL 对象并设置给默认的 buildType DSL。
- 10）、将 whenObjectAdded callbacks 映射到 productFlavor 容器之中。
- 11）、将 whenObjectRemoved 映射在容器之中，当 whenObjectRemoved 回调执行时，会抛出 UnsupportedAction 异常。
- 12）、按顺序依次创建 signingConfig debug、buildType debug、buildType release 类型的 DSL。


其中 **最核心的几项处理可以归纳为如下 四点**：

- 1）、**创建 AppExtension，即 build.gradle 中的 android DSL**。
- 2）、**依次创建应用的 variant 工厂、Task 管理者，variant 管理者**。
- 3）、**注册 新增/移除配置 的 callback，依次包括 signingConfig，buildType，productFlavor**。
- 4）、**依次创建默认的 debug 签名、创建 debug 和 release 两个 buildType**。


在 BasePlugin 的 apply 方法最后，调用了 createTasks 方法来创建 Tasks，该方法如下所示：


```java
 private void createTasks() {
        // 1、在 evaluate 之前创建 Tasks
        threadRecorder.record(
                ExecutionType.TASK_MANAGER_CREATE_TASKS,
                project.getPath(),
                null,
                () -> taskManager.createTasksBeforeEvaluate());

        // 2、创建 Android Tasks
        project.afterEvaluate(
                CrashReporting.afterEvaluate(
                        p -> {
                            sourceSetManager.runBuildableArtifactsActions();

                            threadRecorder.record(
                                    ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                                    project.getPath(),
                                    null,
                                    this::createAndroidTasks);
                        }));
}
```


可以看到，**createTasks 分为两种 Task 的创建方式，即 createTasksBeforeEvaluate 与 createAndroidTasks**。

下面，我们来详细分析下其实现过程。


## 4、TaskManager#createTasksBeforeEvaluate 创建不依赖 flavor 的 task

**TaskManager 的 createTasksBeforeEvaluate 方法给 Task 容器中注册了一系列的 Task，包括 uninstallAllTask、deviceCheckTask、connectedCheckTask、preBuild、extractProguardFiles、sourceSetsTask、assembleAndroidTest、compileLintTask 等等**。


## 5、BasePlugin#createAndroidTasks 创建构建 task

**在 BasePlugin 的 createAndroidTasks 方法中主要
是生成 flavors 相关数据，并根据 flavor 创建与之对应的 Task 实例并注册进 Task 容器之中**。其核心源码如下所示：


```java
 @VisibleForTesting
    final void createAndroidTasks() {
    
    // 1、CompileSdkVersion、插件配置冲突检测（如 JavaPlugin、retrolambda）。
    
    // 创建一些基础或通用的 Tasks。
    
    // 2、将 Project Path、CompileSdk、BuildToolsVersion
    、Splits、KotlinPluginVersion、FirebasePerformancePluginVersion 等信息写入 Project 的配置之中。
    ProcessProfileWriter.getProject(project.getPath())
                .setCompileSdk(extension.getCompileSdkVersion())
                .setBuildToolsVersion(extension.getBuildToolsRevision().toString())
                .setSplits(AnalyticsUtil.toProto(extension.getSplits()));
                
    String kotlinPluginVersion = getKotlinPluginVersion();
        if (kotlinPluginVersion != null) {
            ProcessProfileWriter.getProject(project.getPath())
                    .setKotlinPluginVersion(kotlinPluginVersion);
        }
        AnalyticsUtil.recordFirebasePerformancePluginVersion(project);
                
    // 3、创建应用的 Tasks。
    List<VariantScope> variantScopes = variantManager.createAndroidTasks();

    // 创建一些基础或通用的 Tasks 与做一些通用的处理。
    
}
```


**在 createAndroidTasks 除了创建一些基础或通用的 Tasks 与做一些通用的处理之外， 主要做了三件事**，如下所示：

- 1）、**CompileSdkVersion、插件配置冲突检测（如 JavaPlugin、retrolambda 插件）**。
- 2）、**将 Project Path、CompileSdk、BuildToolsVersion、Splits、KotlinPluginVersion、FirebasePerformancePluginVersion 等信息写入 Project 的配置之中**。
- 3）、**创建应用的 Tasks**。


我们需要 **重点关注 variantManager 的 createAndroidTasks 方法**，去核心源码如下所示：


```java
 /** Variant/Task creation entry point. */
    public List<VariantScope> createAndroidTasks() {
       
        ...
       
        // 1、创建工程级别的测试任务。
        taskManager.createTopLevelTestTasks(!productFlavors.isEmpty());

        // 2、遍历所有 variantScope，为其变体数据创建对应的 Tasks。
        for (final VariantScope variantScope : variantScopes) {
            createTasksForVariantData(variantScope);
        }

        // 3、创建报告相关的 Tasks。
        taskManager.createReportTasks(variantScopes);

        return variantScopes;
    }
```


可以看到，在 createAndroidTasks 方法中有 `三项处理`，如下所示：


- 1）、**创建工程级别的测试任务**。
- 2）、**遍历所有的 variantScope，为其变体数据创建对应的 Tasks**。
- 3）、**创建报告相关的 Tasks**。


接着，我们继续看看 **createTasksForVariantData 方法是如何为每一个指定的 Variant 类型创建对应的 Tasks 的**，其核心源码如下所示：


```java
// 为每一个指定的 Variant 类型创建与之对应的 Tasks
public void createTasksForVariantData(final VariantScope variantScope) {
        final BaseVariantData variantData = variantScope.getVariantData();
        final VariantType variantType = variantData.getType();
        final GradleVariantConfiguration variantConfig = variantScope.getVariantConfiguration();

        // 1、创建 Assemble Task。
        taskManager.createAssembleTask(variantData);
        
        // 2、如果 variantType 是 base moudle，则会创建相应的 bundle Task。需要注意的是，base moudle 是指包含功能的 moudle，而用于 test 的 moudle 则是不包含功能的。
        if (variantType.isBaseModule()) {
            taskManager.createBundleTask(variantData);
        }

        // 3、如果 variantType 是一个 test moudle（其作为一个 test 的组件），则会创建相应的 test variant。
        if (variantType.isTestComponent()) {
        
            // 1）、将 variant-specific, build type multi-flavor、defaultConfig 这些依赖添加到当前的 variantData 之中。
            ...
            
            // 2）、如果支持渲染脚本，则添加渲染脚本的依赖。
            if (testedVariantData.getVariantConfiguration().getRenderscriptSupportModeEnabled()) {
                project.getDependencies()
                        .add(
                                variantDep.getCompileClasspath().getName(),
                                project.files(
                                        globalScope
                                                .getSdkComponents()
                                                .getRenderScriptSupportJarProvider()));
            }
        
            // 3）、如果当前 Variant 会输出一个 APK，即当前是执行的一个 Android test（一般用来进行 UI 自动化测试），则会创建相应的 AndroidTestVariantTask。
            if (variantType.isApk()) { // ANDROID_TEST
                if (variantConfig.isLegacyMultiDexMode()) {
                    String multiDexInstrumentationDep =
                            globalScope.getProjectOptions().get(BooleanOption.USE_ANDROID_X)
                                    ? ANDROIDX_MULTIDEX_MULTIDEX_INSTRUMENTATION
                                    : COM_ANDROID_SUPPORT_MULTIDEX_INSTRUMENTATION;
                    project.getDependencies()
                            .add(
                                    variantDep.getCompileClasspath().getName(),
                                    multiDexInstrumentationDep);
                    project.getDependencies()
                            .add(
                                    variantDep.getRuntimeClasspath().getName(),
                                    multiDexInstrumentationDep);
                }

                taskManager.createAndroidTestVariantTasks(
                        (TestVariantData) variantData,
                        variantScopes
                                .stream()
                                .filter(TaskManager::isLintVariant)
                                .collect(Collectors.toList()));
            } else { // UNIT_TEST
                // 4）、否则说明该 Test moudle 是用于执行单元测试的，则会创建 UnitTestVariantTask。 taskManager.createUnitTestVariantTasks((TestVariantData) variantData);
            }

        } else {
            // 4、如果不是一个 Test moudle，则会调用 ApplicationTaskManager 的 createTasksForVariantScope 方法。
            taskManager.createTasksForVariantScope(
                    variantScope,
                    variantScopes
                            .stream()
                            .filter(TaskManager::isLintVariant)
                            .collect(Collectors.toList()));
        }
}
```


在 createTasksForVariantData 方法中为每一个指定的 Variant 类型创建了与之对应的 Tasks，该方法的处理逻辑如下所示：

- 1、**创建 Assemble Task**。
- 2、**如果 variantType 是 base moudle，则会创建相应的 bundle Task。需要注意的是，base moudle 是指包含功能的 moudle，而用于 test 的 moudle 则是不包含功能的**。
- 3、**如果 variantType 是一个 test moudle（其作为一个 test 的组件），则会创建相应的 test variant**。
    - 1）、**将 variant-specific, build type multi-flavor、defaultConfig 这些依赖添加到当前的 variantData 之中**。
    - 2）、**如果支持渲染脚本，则添加渲染脚本的依赖**。
    - 3）、**如果当前 Variant 会输出一个 APK，即当前是执行的一个 Android test（一般用来进行 UI 自动化测试），则会创建相应的 AndroidTestVariantTask**。
    - 4）、**否则说明该 Test moudle 是用于执行单元测试的，则会创建 UnitTestVariantTask**。
- 4、**如果不是一个 Test moudle，则会调用 ApplicationTaskManager 的 createTasksForVariantScope 方法**。


最终，会执行到 **ApplicationTaskManager 的 createTasksForVariantScope 方法，在这个方法里面创建了适用于应用构建的一系列 Tasks**。 

下面，我们就通过 assembleDebug 的打包流程来分析一下这些 Tasks。


# 六、assembleDebug 打包流程浅析

在对 assembleDebug 构建过程中的一系列 Task 分析之前，我们需要先回顾一下 Android 的打包流程（对这块非常熟悉的同学可以跳过）。


## 1、Android 打包流程回顾

Android 官方的编译打包流程图如下所示：

![](https://user-gold-cdn.xitu.io/2020/3/30/1712a17ebc47801d?w=950&h=1068&f=png&s=61290)

比较粗略的打包流程可简述为如下 **四个步骤**：


- 1）、**编译器会将 APP 的源代码转换成 DEX（Dalvik Executable) 文件（其中包括 Android 设备上运行的字节码），并将所有其他内容转换成已编译资源**。
- 2）、**APK 打包器将 DEX 文件和已编译的资源合并成单个 APK。 但是，必须先签署 APK，才能将应用安装并部署到 Android 设备上**。
- 3）、**APK 打包器会使用相应的 keystore 发布密钥库去签署 APK**。
- 4）、**在生成最终的 APK 之前，打包器会使用 zipalign 工具对应用进行优化，减少其在设备上运行时占用的内存**。


为了 **了解更多打包过程中的细节**，我们需要查看更加详细的旧版 APK 打包流程图 ，如下图所示：

![](https://user-gold-cdn.xitu.io/2020/3/25/1710fc302da5fd87?imageslim)

比较详细的打包流程可简述为如下 **八个步骤**：

- 1、**首先，.aidl（Android Interface Description Language）文件需要通过 aidl 工具转换成编译器能够处理的 Java 接口文件**。
- 2、**同时，资源文件（包括 AndroidManifest.xml、布局文件、各种 xml 资源等等）将被 AAPT（Asset Packaging Tool）（Android Gradle Plugin 3.0.0 及之后使用 AAPT2 替代了 AAPT）处理为最终的 resources.arsc，并生成 R.java 文件以保证源码编写时可以方便地访问到这些资源**。
- 3、**然后，通过 Java Compiler 编译 R.java、Java 接口文件、Java 源文件，最终它们会统一被编译成 .class 文件**。
- 4、**因为 .class 并不是 Android 系统所能识别的格式，所以还需要通过 dex 工具将它们转化为相应的 Dalvik 字节码（包含压缩常量池以及清除冗余信息等工作）。这个过程中还会加入应用所依赖的所有 “第三方库”**。
- 5、**下一步，通过 ApkBuilder 工具将资源文件、DEX 文件打包生成 APK 文件**。
- 6、**接着，系统将上面生成的 DEX、资源包以及其它资源通过 apkbuilder 生成初始的 APK 文件包**。
- 7、**然后，通过签名工具 Jarsigner 或者其它签名工具对 APK 进行签名得到签名后的 APK。如果是在 Debug 模式下，签名所用的 keystore 是系统自带的默认值，否则我们需要提供自己的私钥以完成签名过程**。
- 8、**最后，如果是正式版的 APK，还会利用 ZipAlign 工具进行对齐处理，以提高程序的加载和运行速度。而对齐的过程就是将 APK 文件中所有的资源文件距离文件的起始位置都偏移4字节的整数倍，这样通过 mmap 访问 APK 文件的速度会更快，并且会减少其在设备上运行时的内存占用**。

至此，我们已经了解了整个 **APK** 编译和打包的流程。


### 那么，为什么 XML 资源文件要从文本格式编译成二进制格式？

主要基于以下 `两点原因`：

- 1、`空间占用更小`：**因为所有 XML 元素的标签、属性名称、属性值和内容所涉及到的字符串都会被统一收集到一个字符串资源池中，并且会去重。有了这个字符串资源池，原来使用字符串的地方就会被替换成一个索引到字符串资源池的整数值，从而可以减少文件的大小**。

- 2、`解析效率更高`：**二进制格式的 XML 文件解析速度更快。这是由于二进制格式的 XML 元素里面不再包含有字符串值，因此就避免了进行字符串解析，从而提高了解析效率**。


### 而 Android 资源管理框架又是如何快速定位到最匹配资源的？

主要基于两个文件，如下所示：

- 1、`资源 ID 文件 R.java`：**赋予每一个非 assets 资源一个 ID 值，这些 ID 值以常量的形式定义在 R.java 文件中**。
- 2、`资源索引表 resources.arsc`：**用来描述那些具有 ID 值的资源的配置信息**。


## 2、assmableDebug 打包流程浅析

我们可以通过下面的命令来获取打包一个 Debug APK 所需要的执行的 Task，如下所示：


```gradle
quchao@quchaodeMacBook-Pro CustomPlugin % ./gradlew app:assembleDebug --console=plain

...

> Task :app:preBuild UP-TO-DATE
> Task :app:preDebugBuild UP-TO-DATE
> Task :app:generateDebugBuildConfig
> Task :app:javaPreCompileDebug
> Task :app:mainApkListPersistenceDebug
> Task :app:generateDebugResValues
> Task :app:createDebugCompatibleScreenManifests
> Task :app:extractDeepLinksDebug
> Task :app:compileDebugAidl NO-SOURCE
> Task :app:compileDebugRenderscript NO-SOURCE
> Task :app:generateDebugResources
> Task :app:processDebugManifest
> Task :app:mergeDebugResources
> Task :app:processDebugResources
> Task :app:compileDebugJavaWithJavac
> Task :app:compileDebugSources
> Task :app:mergeDebugShaders
> Task :app:compileDebugShaders
> Task :app:generateDebugAssets
> Task :app:mergeDebugAssets
> Task :app:processDebugJavaRes NO-SOURCE
> Task :app:checkDebugDuplicateClasses
> Task :app:dexBuilderDebug
> Task :app:mergeLibDexDebug
> Task :app:mergeDebugJavaResource
> Task :app:mergeDebugJniLibFolders
> Task :app:validateSigningDebug
> Task :app:mergeProjectDexDebug
> Task :app:mergeDebugNativeLibs
> Task :app:stripDebugDebugSymbols
> Task :app:desugarDebugFileDependencies
> Task :app:mergeExtDexDebug
> Task :app:packageDebug
> Task :app:assembleDebug
```


在 TaskManager 中，**主要有两种方法用来去创建 Task**，它们分别为 `createTasksBeforeEvaluate` 方法与 `createTasksForVariantScope` 方法。需要注意的是，**createTasksForVariantScope 方法是一个抽象方法，其具体的创建 Tasks 的任务分发给了 TaskManager 的子类进行处理，其中最常见的子类要数 ApplicationTaskManager 了，它就是在 Android 应用程序中用于创建 Tasks 的 Task 管理者**。

其中，打包流程中的大部分 tasks 都在这个目录之下：

> com.android.build.gradle.internal.tasks


下面，我们看看 assembleDebug 打包流程中所需的各个 Task 所对应的实现类与含义，如下表所示：

| Task                                                 | 对应实现类                  | 作用                                                                 |
| ---------------------------------------------------- | --------------------------- | -------------------------------------------------------------------- |
|     preBuild                       |        AppPreBuildTask                                        |   预先创建的 task，用于做一些 application Variant 的检查
| preDebugBuild                                        |                             | 与 preBuild 区别是这个 task 是用于在 Debug 的环境下的一些 Vrariant 检查 |
| generateDebugBuildConfig                             | GenerateBuildConfig         | 生成与构建目标相关的 BuildConfig 类                                              |
| javaPreCompileDebug                                        |    JavaPreCompileTask                         | 用于在 Java 编译之前执行必要的 action |
| mainApkListPersistenceDebug                                        |   MainApkListPersistence                          | 用于持久化 APK 数据 |
| generateDebugResValues                                        |    GenerateResValues                         | 生成 Res 资源类型值 |
| createDebugCompatibleScreenManifests                                        |                      CompatibleScreensManifest       | 生成具有给定屏幕密度与尺寸列表的 <compatible-screens> （兼容屏幕）节点清单 |
| extractDeepLinksDebug                                        |      ExtractDeepLinksTask                       | 用于抽取一系列 DeepLink（深度链接技术，主要应用场景是通过Web页面直接调用Android原生app，并且把需要的参数通过Uri的形式，直接传递给app，节省用户的注册成本）  |
| compileDebugAidl                                     | AidlCompile                 | 编译 AIDL 文件                                         |
| compileDebugRenderscript                             | RenderscriptCompile         | 编译 Renderscript 文件                                                    |
| generateDebugResources                               |     在 TaskManager.createAnchorTasks 方法中通过 taskFactory.register(taskName)的方式注册一个 task                        | 空 task，锚点                                                        |
| processDebugManifest                                 | ProcessApplicationManifest              | 处理 manifest 文件                                                   |
| mergeDebugResources                                  | MergeResources              | 使用 AAPT2 合并资源文件                                                         |
| processDebugResources                                | ProcessAndroidResources     | 用于处理资源并生成 R.class 文件                                                      |
| compileDebugJavaWithJavac                            | JavaCompileCreationAction（这里是一个 Action，从 gradle 源码中可以看到从 TaskFactory 中注册一个 Action 可以得到与之对应的 Task，因此，Task 即 Action，Action 即 Task）          | 用于执行 Java 源码的编译                                                       |
| compileDebugSources                                  |    在 TaskManager.createAnchorTasks 方法中通过 taskFactory.register(taskName)的方式注册一个 task                        | 空 task，锚点使用                                                    |
| mergeDebugShaders                                    | MergeSourceSetFolders.MergeShaderSourceFoldersCreationAction       | 合并 Shader 文件                                                     |
| compileDebugShaders                                  | ShaderCompile               | 编译 Shaders                                                         |
| generateDebugAssets                                  |     在 TaskManager.createAnchorTasks 方法中通过 taskFactory.register(taskName)的方式注册一个 task                        | 空 task，锚点                                                        |
| mergeDebugAssets                                     | MergeSourceSetFolders.MergeAppAssetCreationAction       | 合并 assets 文件                                                     |
| processDebugJavaRes                                  | ProcessJavaResConfigAction  | 处理 Java Res 资源                                                      |
| checkDebugDuplicateClasses           |  CheckDuplicateClassesTask |                             用于检测工程外部依赖，确保不包含重复类                            |
| dexBuilderDebug           | DexArchiveBuilderTask |      用于将 .class 文件转换成 dex archives，即 DexArchive，Dex 存档，可以通过 addFile 添加一个 DEX 文件                                                   |
| mergeLibDexDebug           | DexMergingTask.DexMergingAction.MERGE_LIBRARY_PROJECT |              仅仅合并库工程中的 DEX 文件                                           |
| mergeDebugJavaResource           | MergeJavaResourceTask |    合并来自多个 moudle 的 Java 资源                                                     |
| mergeDebugJniLibFolders           | MergeSourceSetFolders.MergeJniLibFoldersCreationAction |     以合适的优先级合并 JniLibs 源文件夹                                                    |
| validateSigningDebug           | ValidateSigningTask |      用于检查当前 Variant 的签名配置中是否存在密钥库文件，如果当前密钥库默认是 debug keystore，即使它不存在也会进行相应的创建                                                 |
| mergeProjectDexDebug           | DexMergingTask.DexMergingAction.MERGE_PROJECT |              仅仅合并工程的 DEX 文件                                                        |
| mergeDebugNativeLibs           | MergeNativeLibsTask |          从多个 moudle 中合并 native 库                                               |
| stripDebugDebugSymbols           | StripDebugSymbolsTask |   从 Native 库中移除 Debug 符号。                                                      |
| desugarDebugFileDependencies           | DexFileDependenciesTask |                            处理 Dex 文件的依赖关系                             |
| mergeExtDexDebug           | DexMergingTask.DexMergingAction.MERGE_EXTERNAL_LIBS |      仅仅用于合并外部库的 DEX 文件                                                   |
| packageDebug                                         | PackageApplication          | 打包 APK                                                             |
| assembleDebug                                        |    Assemble                         | 空 task，锚点使用                                                     |


目前，**在 Gradle Plugin 中主要有三种类型的 Task**，如下所示：

- 1）、`增量 Task`：**继承于 NewIncrementalTask 这个增量 Task 基类，需要重写 doTaskAction 抽象方法实现增量功能**。
- 2）、`非增量 Task`：**继承于 NonIncrementalTask 这个非增量 Task 基类，重写 doTaskAction 抽象方法实现全量更新功能**。
- 3）、`Transform Task`：**我们编写的每一个自定义 Transform 会在调用 appExtension.registerTransform(new CustomTransform()) 注册方法时将其保存到当前的 Extension 类中的 transforms 列表中，当 LibraryTaskManager/TaskManager 调用 createPostCompilationTasks（负责为给定 Variant 创建编译后的 task）方法时，会取出相应 Extension 中的 tranforms 列表进行遍历，并通过 TransformManager.addTransform 方法将每一个 Transform 转换为与之对应的 TransformTask 实例，而该方法内部具体是通过 new TransformTask.CreationAction(...) 的形式进行创建**。


全面了解了打包过程中涉及到的一系列 Tasks 与 Task 必备的一些基础知识之后，我们再来对其中最重要的几个 Task 的实现来进行详细分析。


# 七、重要 Task 实现源码分析

## 1、资源处理相关 Task

### 1）、processDebugManifest

processDebugManifest 对应的实现类为 ProcessApplicationManifest Task，它继承了 IncrementalTask，但是没有实现 isIncremental 方法，因此我们只需看其 doFullTaskAction 方法即可。   


#### 调用链路


```java
processDebugManifest.dofFullTaskAction => ManifestHelperKt.mergeManifestsForApplication => ManifestMerge2.merge
```

#### 主要流程分析

这个 task 功能主要是 **用于合并所有的（包括 module 和 flavor） mainfest，其过程主要是利用 MergingReport，ManifestMerger2 和 XmlDocument 这三个实例进行处理**。

我们直接关注到 ManifestMerger2.merge 方法的 merge 过程，看看具体的合并是怎样的。其主体步骤如下所示：

##### 1、获取主 manifest 的信息，以做一些必要的检查，这里会返回一个 LoadedManifestInfo 实例。

```java
// load the main manifest file to do some checking along the way.
LoadedManifestInfo loadedMainManifestInfo =
        load(
                new ManifestInfo(
                        mManifestFile.getName(),
                        mManifestFile,
                        mDocumentType,
                        Optional.absent() /* mainManifestPackageName*/),
                selectors,
                mergingReportBuilder);
```


##### 2、执行 Manifest 中的系统属性注入：将主 Manifest 中定义的某些属性替换成 gradle 中定义的属性，例如 package, version_code, version_name, min_sdk_versin 、target_sdk_version、max_sdk_version 等等。

``` java
// perform system property injection
performSystemPropertiesInjection(mergingReportBuilder,
        loadedMainManifestInfo.getXmlDocument());
```

```java
/**
 * Perform {@link ManifestSystemProperty} injection.
 * @param mergingReport to log actions and errors.
 * @param xmlDocument the xml document to inject into.
 */
protected void performSystemPropertiesInjection(
        @NonNull MergingReport.Builder mergingReport,
        @NonNull XmlDocument xmlDocument) {
    for (ManifestSystemProperty manifestSystemProperty : ManifestSystemProperty.values()) {
        String propertyOverride = mSystemPropertyResolver.getValue(manifestSystemProperty);
        if (propertyOverride != null) {
            manifestSystemProperty.addTo(
                    mergingReport.getActionRecorder(), xmlDocument, propertyOverride);
        }
    }
}
```


##### 3、合并 flavors 并且构建与之对应的 manifest 文件。

```groovy
for (File inputFile : mFlavorsAndBuildTypeFiles) {
    mLogger.verbose("Merging flavors and build manifest %s \n", inputFile.getPath());
    LoadedManifestInfo overlayDocument =
            load(
                    new ManifestInfo(
                            null,
                            inputFile,
                            XmlDocument.Type.OVERLAY,
                            mainPackageAttribute.transform(it -> it.getValue())),
                    selectors,
                    mergingReportBuilder);
    if (!mFeatureName.isEmpty()) {
        overlayDocument =
                removeDynamicFeatureManifestSplitAttributeIfSpecified(
                        overlayDocument, mergingReportBuilder);
    }
    // 1、检查 package 定义
    Optional<XmlAttribute> packageAttribute =
            overlayDocument.getXmlDocument().getPackage();
    // if both files declare a package name, it should be the same.
    if (loadedMainManifestInfo.getOriginalPackageName().isPresent() &&
            packageAttribute.isPresent()
            && !loadedMainManifestInfo.getOriginalPackageName().get().equals(
            packageAttribute.get().getValue())) {
        // 2、如果 package 定义重复的话，会输出下面信息
        String message = mMergeType == MergeType.APPLICATION
                ? String.format(
                        "Overlay manifest:package attribute declared at %1$s value=(%2$s)\n"
                                + "\thas a different value=(%3$s) "
                                + "declared in main manifest at %4$s\n"
                                + "\tSuggestion: remove the overlay declaration at %5$s "
                                + "\tand place it in the build.gradle:\n"
                                + "\t\tflavorName {\n"
                                + "\t\t\tapplicationId = \"%2$s\"\n"
                                + "\t\t}",
                        packageAttribute.get().printPosition(),
                        packageAttribute.get().getValue(),
                        mainPackageAttribute.get().getValue(),
                        mainPackageAttribute.get().printPosition(),
                        packageAttribute.get().getSourceFile().print(true))
                : String.format(
                        "Overlay manifest:package attribute declared at %1$s value=(%2$s)\n"
                                + "\thas a different value=(%3$s) "
                                + "declared in main manifest at %4$s",
                        packageAttribute.get().printPosition(),
                        packageAttribute.get().getValue(),
                        mainPackageAttribute.get().getValue(),
                        mainPackageAttribute.get().printPosition());
        mergingReportBuilder.addMessage(
                overlayDocument.getXmlDocument().getSourceFile(),
                MergingReport.Record.Severity.ERROR,
                message);
        return mergingReportBuilder.build();
    }
    ...
}
```


##### 4、合并库中的 manifest 文件


``` groovy
for (LoadedManifestInfo libraryDocument : loadedLibraryDocuments) {
    mLogger.verbose("Merging library manifest " + libraryDocument.getLocation());
    xmlDocumentOptional = merge(
            xmlDocumentOptional, libraryDocument, mergingReportBuilder);
    if (!xmlDocumentOptional.isPresent()) {
        return mergingReportBuilder.build();
    }
}
```


##### 5、执行 manifest 文件中的 placeholder 替换


``` groovy
performPlaceHolderSubstitution(
        loadedMainManifestInfo,
        xmlDocumentOptional.get(),
        mergingReportBuilder,
        severity);
```


##### 6、之后对最终合并后的 manifest 中的一些属性进行一次替换，与步骤 2 类似。

##### 7、保存 manifest 到 build/intermediates/merged_manifests/flavorName/AndroidManifest.xml，至此，已生成最终的 Manifest 文件。


### 2）、mergeDebugResources  

mergeDebugResources 对应的是 MergeResources Task，它 **使用了 AAPT2 合并资源**。


#### 调用链路


```java
MergeResources.doFullTaskAction => ResourceMerger.mergeData => MergedResourceWriter.end => mResourceCompiler.submitCompile => AaptV2CommandBuilder.makeCompileCommand
```

#### 主体流程分析

MergeResources 继承自 IncrementalTask，对于 增量 Task 来说我们只需看如下三个方法的实现：

- `isIncremental`
- `doFullTaskAction`
- `doIncrementalTaskAction`


##### 1、首先查看 isIncremental 方法。


``` groovy
    // 说明 MergeResources Task 支持增量，肯定重写了 doIncrementalTaskAction 方法
    protected boolean isIncremental() {
        return true;
    }
```


##### 2、然后，查看 doFullTaskAction 方法，内部通过 getConfiguredResourceSets 方法获取了 resourceSets，包括了自己的 res 和依赖库的 res 资源以及 build/generated/res/rs。


``` groovy
List<ResourceSet> resourceSets = getConfiguredResourceSets(preprocessor);
```


##### 3、创建 ResourceMerger，并使用 resourceSets 进行填充。


``` groovy
ResourceMerger merger = new ResourceMerger(minSdk.get());
```


##### 4、创建 ResourceCompilationService，它使用了 aapt2。


``` groovy
// makeAapt 中使用 aapt2，然后返回 ResourceCompilationService 实例.
ResourceCompilationService resourceCompiler =
        getResourceProcessor(
                getAapt2FromMaven(),
                workerExecutorFacade,
                errorFormatMode,
                flags,
                processResources,
                getLogger())) {
```


##### 5、将第 2 步获取的 resourceSet 加入至 ResourceMerger 中。


``` groovy
for (ResourceSet resourceSet : resourceSets) {
    resourceSet.loadFromFiles(new LoggerWrapper(getLogger()));
    merger.addDataSet(resourceSet);
}
```


##### 6、创建 MergedResourceWriter

```groovy
MergedResourceWriter writer =
        new MergedResourceWriter(
                workerExecutorFacade,
                destinationDir,
                publicFile,
                mergingLog,
                preprocessor,
                resourceCompiler,
                getIncrementalFolder(),
                dataBindingLayoutProcessor,
                mergedNotCompiledResourcesOutputDirectory,
                pseudoLocalesEnabled,
                getCrunchPng());
```


##### 7、调用 ResourceMerger.mergeData 方法对资源进行合并。


``` groovy
merger.mergeData(writer, false /*doCleanUp*/);
```


##### 8、调用 MergedResourceWriter 的 start，ignoreItemInMerge、removeItem、addItem，end 方法，其中 item 中包括了需要处理的资源，包括 xml 和 图片资源，每一个 item 对应的文件，都会创建一个与之对应的 CompileResourceRequest 实例，并加入到 mCompileResourceRequests 这个 ConcurrentLinkedQueue 队列中。


##### 9、调用 mResourceCompiler.submitCompile 方法处理资源。

```java
// MergedResourceWriter.end()
mResourceCompiler.submitCompile(
        new CompileResourceRequest(
                fileToCompile,
                request.getOutputDirectory(),
                request.getInputDirectoryName(),
                request.getInputFileIsFromDependency(),
                pseudoLocalesEnabled,
                crunchPng,
                ImmutableMap.of(),
                request.getInputFile()));
mCompiledFileMap.put(
        fileToCompile.getAbsolutePath(),
        mResourceCompiler.compileOutputFor(request).getAbsolutePath());
```       
  

在 submitCompile 中最终会使用 AaptV2CommandBuilder.makeCompileCommand 方法生成 aapt2 命令去处理资源。


##### 10、最后，对 doIncrementalTaskAction 的实现我这里就不赘述了，因为增量 task 的实现过程和全量实现差异不大，仅仅是使用修改后的文件去获取 resourceSets 。


## 2、将 Class 文件打包成 Dex 文件的过程

即 DexArchiveBuilderTask，它具体对应的是 DexArchiveBuilderTask，用于将 .class 文件转换成 dex archives，即 Dex 存档，它可以通过 addFile 添加一个 DEX 文件。


### 调用链路

```Java
DexArchiveBuilderTask.doTaskAction => DexArchiveBuilderTaskDelegate.doProcess => DexArchiveBuilderTaskDelegate.processClassFromInput => DexArchiveBuilderTaskDelegate.convertToDexArchive -> DexArchiveBuilderTaskDelegate.launchProcessing -> DexArchiveBuilder.convert
```


#### 主体流程分析

在 DexArchiveBuilderTask 中，**对 class 的处理方式分为两种，一种是对 目录下的 class 进行处理，一种是对 .jar 里面的 class 进行处理**。   

##### 那么，这里为什么要分为这两种方式呢？

**因为 .jar 中的 class 文件通常来说都是依赖库，基本上不会改变，所以 gradle 在这里就可以实现一个缓存操作**。


##### 1、convertJarToDexArchive 处理 jar   

在处理 jar 包的时候，Gradle 会对 jar 包中的每一个 class 文件都单独打成一个 DEX 文件，然后再把它们放回 jar 包之中。



```groovy
private fun convertJarToDexArchive(
    jarInput: File,
    outputDir: File,
    bootclasspath: ClasspathServiceKey,
    classpath: ClasspathServiceKey,
    cacheInfo: D8DesugaringCacheInfo
): List<File> {
    if (cacheInfo !== DesugaringDontCache) {
        val cachedVersion = cacheHandler.getCachedVersionIfPresent(
            jarInput, cacheInfo.orderedD8DesugaringDependencies
        )
        if (cachedVersion != null) {
            // 如果有缓存，直接使用缓存的 jar 包。
            val outputFile = getOutputForJar(jarInput, outputDir, null)
            Files.copy(
                cachedVersion.toPath(),
                outputFile.toPath(),
                StandardCopyOption.REPLACE_EXISTING
            )
            // no need to try to cache an already cached version.
            return listOf()
        }
    }
    // 如果没有缓存，则调用 convertToDexArchive 方法去生成 dex。
    return convertToDexArchive(
        jarInput,
        outputDir,
        false,
        bootclasspath,
        classpath,
        setOf(),
        setOf()
    )
}
```


##### 2、使用 convertToDexArchive 处理 dir 以及 jar 的后续处理     

内部会调用 launchProcessing 对 dir 进行处理，代码如下所示：


``` groovy
private fun launchProcessing(
    dexConversionParameters: DexArchiveBuilderTaskDelegate.DexConversionParameters,
    outStream: OutputStream,
    errStream: OutputStream,
    receiver: MessageReceiver
) {
    val dexArchiveBuilder = dexConversionParameters.getDexArchiveBuilder(
        outStream,
        errStream,
        receiver
    )
    val inputPath = dexConversionParameters.input.toPath()
    val hasIncrementalInfo =
        dexConversionParameters.input.isDirectory && dexConversionParameters.isIncremental
     // 如果 class 新增 || 修改过，就进行处理    
    fun toProcess(path: String): Boolean {
        if (!dexConversionParameters.belongsToThisBucket(path)) return false
        if (!hasIncrementalInfo) {
            return true
        }
        val resolved = inputPath.resolve(path).toFile()
        return resolved in dexConversionParameters.additionalPaths || resolved in dexConversionParameters.changedFiles
    }
    val bucketFilter = { name: String -> toProcess(name) }
    loggerWrapper.verbose("Dexing '" + inputPath + "' to '" + dexConversionParameters.output + "'")
    try {
        ClassFileInputs.fromPath(inputPath).use { input ->
            input.entries(bucketFilter).use { entries ->
                // 内部会调用 dx || d8 去生成 dex 文件
                dexArchiveBuilder.convert(
                    entries,
                    Paths.get(URI(dexConversionParameters.output)),
                    dexConversionParameters.input.isDirectory
                )
            }
        }
    } catch (ex: DexArchiveBuilderException) {
        throw DexArchiveBuilderException("Failed to process $inputPath", ex)
    }
}
```


可以看到，在 DexArchiveBuilder 有两个子类，它们分别如下所示：

- `D8DexArchiveBuilder`：**调用 D8 去生成 DEX 文件**。
- `DxDexArchiveBuilder`：**调用 DX 去生成 DEX 文件**。


我们这里就以 D8DexArchiveBuilder 为例来说明其是如何调用 D8 去生成 DEX 文件的。其源码如下所示：


```java
@Override
public void convert(
        @NonNull Stream<ClassFileEntry> input, @NonNull Path output, boolean isIncremental)
        throws DexArchiveBuilderException {
    // 1、创建一个 D8 诊断信息处理器实例，用于发出不同级别的诊断信息，共分为三类，由严重程度递减分别为：error、warning、info。
    D8DiagnosticsHandler d8DiagnosticsHandler = new InterceptingDiagnosticsHandler();
    try {
        // 2、创建一个 D8 命令构建器实例。
        D8Command.Builder builder = D8Command.builder(d8DiagnosticsHandler);
        AtomicInteger entryCount = new AtomicInteger();
        
        // 3、遍历读取每一个类的字节数据。
        input.forEach(
                entry -> {
                    builder.addClassProgramData(
                            readAllBytes(entry), D8DiagnosticsHandler.getOrigin(entry));
                    entryCount.incrementAndGet();
                });
        if (entryCount.get() == 0) {
            // 3、如果没有可遍历的数据，则直接 return。这里使用 AtomicInteger 类来实现了是否有遍历了数据的区分处理。
            return;
        }
        OutputMode outputMode =
                isIncremental ? OutputMode.DexFilePerClassFile : OutputMode.DexIndexed;
                
        // 4、给 D8 命令构建器实例设置一系列的配置，例如 编译模式、最小 Sdk 版本等等。
        builder.setMode(compilationMode)
                .setMinApiLevel(minSdkVersion)
                .setIntermediate(true)
                .setOutput(output, outputMode)
                .setIncludeClassesChecksum(compilationMode == compilationMode.DEBUG);
        if (desugaring) {
            builder.addLibraryResourceProvider(bootClasspath.getOrderedProvider());
            builder.addClasspathResourceProvider(classpath.getOrderedProvider());
            if (libConfiguration != null) {
                builder.addSpecialLibraryConfiguration(libConfiguration);
            }
        } else {
            builder.setDisableDesugaring(true);
        }
        
        // 5、使用 com.android.tools.r8 工具包中的 D8 类的 run 方法运行组装后的 D8 命令。
        D8.run(builder.build(), MoreExecutors.newDirectExecutorService());
    } catch (Throwable e) {
        throw getExceptionToRethrow(e, d8DiagnosticsHandler);
    }
}
```


D8DexArchiveBuilder 的 convert 过程可以归纳为 `五个步骤`，如下所示：

- 1）、**创建一个 D8 诊断信息处理器实例，用于发出不同级别的诊断信息，共分为三类，由严重程度递减分别为：error、warning、info**。
- 2）、**创建一个 D8 命令构建器实例**。
- 3）、**遍历读取每一个类的字节数据**。
- 4）、**给 D8 命令构建器实例设置一系列的配置，例如 编译模式、最小 Sdk 版本等等**。
- 5）、**使用 com.android.tools.r8 工具包中的 D8 类的 run 方法运行组装后的 D8 命令**。


# 八、总结

我们再回头看看开篇时的那一幅 **Gradle 插件的整体实现架构图**，如下所示：


![](https://user-gold-cdn.xitu.io/2020/4/27/171ba83821461652?w=434&h=580&f=png&s=38095)


## 最后的最后

我们可以 **根据上面这幅图，由下而上细细地思考回忆一下，每一层的主要流程是什么？其中涉及的一些关键细节具体有哪些？此时，你是否觉得已经真正地理解了 Gradle 插件架构的实现原理呢**？


# 参考链接：
---
1、Android Gradle Plugin V3.6.2 源码

2、Gradle V5.6.4 源码

3、[Android Plugin DSL Reference](http://google.github.io/android-gradle-dsl/current/)

4、[Gradle DSL Reference](https://docs.gradle.org/current/dsl/index.html)

5、[designing-gradle-plugins](https://guides.gradle.org/designing-gradle-plugins/)

6、[android-training => gradle](https://github.com/5A59/android-training/tree/master/gradle)

7、[连载 | 深入理解Gradle框架之一：Plugin, Extension, buildSrc](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247485042&idx=1&sn=fe32711dbcb483f7a47dfa0e304087c4&scene=21#wechat_redirect)

8、[连载 | 深入理解gradle框架之二：依赖实现分析](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247485061&idx=1&sn=7c935aecbb9b558e4d5d9dd2c3eb7f96&scene=21#wechat_redirect)

9、[连载 | 深入理解gradle框架之三：artifacts的发布](https://mp.weixin.qq.com/s/-G0OHJrNHbjXTC7AHtVAKw)

10、[Android Gradle Plugin 源码解析（上）](https://mp.weixin.qq.com/s?__biz=MzIwMTAzMTMxMg==&mid=2649492752&idx=1&sn=1d1ad65c63667d96b72452a492cbde58&chksm=8eec86efb99b0ff9378916d061f70d56bc1c27c795ef5c36cd5692886c3cbb1109636951a5c3&scene=38#wechat_redirect)

11、[Android Gradle Plugin 源码解析（下）](https://mp.weixin.qq.com/s?__biz=MzIwMTAzMTMxMg==&mid=2649492778&idx=1&sn=bf18cd9b7e4fb5f08b1a698836807304&chksm=8eec86d5b99b0fc30d7a85f4b4d0ffe4bbc4d00b5b5e89f0ef005048a5ce4c030f1aad74f69f&scene=38#wechat_redirect)

12、[Gradle 庖丁解牛（构建源头源码浅析）](https://blog.csdn.net/yanbober/article/details/60584621)

13、[Gradle 庖丁解牛（构建生命周期核心委托对象创建源码浅析）](https://blog.csdn.net/yanbober/article/details/65635040)


# Contanct Me

##  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

##  ●  微信群：

> **由于微信群已超过 200 人，麻烦大家想进微信群的朋友们，加我微信拉你进群。**


##  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


## About me

- ### Email: [chao.qu521@gmail.com]()
- ### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- ### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。

















