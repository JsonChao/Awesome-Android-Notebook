---

		title:  深入探索Gradle自动化构建技术（一、全面掌握Gradle配置）
		date: 2020/2/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---
# 前言

### 成为一名优秀的Android开发，需要一份完备的 [知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


# 一、重识 Gradle

**工程构建工具从古老的 mk、make、cmake、qmake, 再到成熟的 ant、maven、ivy，最后到如今互联网时代的 sbt、gradle，经历了长久的历史演化与变迁**。

Gradle 作为一款新生代的构建工具无疑是有它自身的巨大优势的，**因此，掌握好 Gradle 构建工具的各种使用姿势与使用场景其重要性不言而喻**。

此外，**Gradle** 已经成为 **高级 Android 知识体系** 必不可少的一部分。因此，掌握 Gradle，提升自身 **自动化构建技术的深度**， 能让我们更加地 **如虎添翼**。

## 1、Gradle 是什么？

- 1)、<span style="color:#0e88eb;font-weight:bold;">它是一款强大的构建工具，而不是语⾔。</span>
- 2)、<span style="color:#0e88eb;font-weight:bold;">它使用了 Groovy 这个语言，创造了一种 DSL，但它本身不是语⾔。</span>

## 2、为什么使用 Gradle?

主要基于如下 <span style="color:#773098;font-weight:bold;">三点</span> 原因：

- 1）、<span style="color:#0e88eb;font-weight:bold;">它是一个款最新的，功能最强大的构建工具，使用它我们能做很多事情。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">使用程序替代传统的 XML 配置，使得项目构建更加灵活。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">丰富的第三方插件，可以让我们随心所欲地使用。</span>

## 3、Gradle 的构建流程

通常来说，Gradle 一次完整的构建过程通常分成如下 <span style="color:#773098;font-weight:bold;">三个部分</span>：

- <span style="color:rgb(248,57,41);font-weight:bold;">初始化阶段：</span><span style="color:#0e88eb;font-weight:bold;">首先，在初始化阶段 Gradle 会决定哪些项目模块要参与构建，并且为每个项目模块创建一个与之对应的 Project 实例。</span>
- <span style="color:rgb(248,57,41);font-weight:bold;">配置阶段：</span><span style="color:#0e88eb;font-weight:bold;">然后，配置工程中每个项目的模块，并执行包含其中的配置脚本。</span>
- <span style="color:rgb(248,57,41);font-weight:bold;">任务执行：</span><span style="color:#0e88eb;font-weight:bold;">最后，执行每个参与构建过程的 Gradle task。</span>


# 二、打包提速

掌握 Gradle 构建提速的技巧能够帮助我们节省大量的编译构建时间，并且，依赖模块越多且越大的项目节省出来的时间越多，因此是一件投入产出比相当大的事情。

## 1、升级最新的 Gradle 版本

将 Gradle 和 Android Gradle Plugin 的版本升至最新，所带来的的构建速度的提升效果是显而易见的，特别是当之前你所使用的版本很低的时候。


## 2、开启离线模式

打开 Android Studio 的离线模式后，所有的编译操作都会走本地缓存，毫无疑问，这将会极大地缩短编译时间。


## 3、配置 AS 的最大堆内存

**在默认情况下， AS 的最大堆内存为 1960MB，我们可以选择 Help => Edit Custom VM Options，此时，会打开一个 studio.vmoptions 文件，我们将第二行的 -Xmx1960m 改为 -Xmx3g 即可将可用内存提升到 3GB**。


## 4、删除不必要的 Moudle 或合并部分 Module

过多的 Moudle 会使项目中 Module 的依赖关系变得复杂，Gradle 在编译构建的时候会去检测各个 Module 之间的依赖关系，然后，它会花费大量的构建时间帮我们梳理这些 Module 之间的依赖关系，以避免 Module 之间相互引用而带来的各种问题。**除了删除不必要的 Moudle 或合并部分 Module 的方式外，我们也可以将稳定的底层 Module 打包成 aar，上传到公司的本地 Maven 仓库，通过远程方式依赖**。


## 5、删除Module中的无用文件

- 1）、<span style="color:#0e88eb;font-weight:bold;">如果我们不需要写单元测试代码，可以直接删除 test 目录。</span>
- 2）、<span style="color:#0e88eb;font-weight:bold;">如果我们不需要写 UI 测试代码，也可以直接删除 androidTest 目录。</span>
- 3）、<span style="color:#0e88eb;font-weight:bold;">此外，如果 Moudle 中只有纯代码，可以直接删除 res 目录。</span>


## 6、去除项目中的无用资源

在 Android Studio 中提供了供了自动检测失效文件和删除的功能，即 **Remove Unused Resource** 功能，操作路径如下所示：

> 右键 => 选中 Refactor => 选中Remove Unused Resource => 直接点击REFACTOR
    
    
**需要注意的是，这里不需要将 Delete unused @id declarations too 选中，如果你使用了 databinding 的话，可能会编译失败**。


## 7、优化第三方库的使用

一般的优化步骤有如下 <span style="color:#773098;font-weight:bold;">三步：</span>

### 1）、使用更小的库去替换现有的同类型的三方库。

### 2）、使用 exclude 来排除三方库中某些不需要或者是重复的依赖。

例如，我在 [Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid) 项目中就使用到了这种技巧，在依赖 LeakCanary 时，发现它包含有 support 包，因此，我们可以使用 exclude 将它排除掉，代码如下所示：

 
 ```java
    debugImplementation (rootProject.ext.dependencies["leakcanary-android"]) {
        exclude group: 'com.android.support'
    }
    releaseImplementation (rootProject.ext.dependencies["leakcanary-android-no-op"]) {
        exclude group: 'com.android.support'
    }
    testImplementation (rootProject.ext.dependencies["leakcanary-android-no-op"]) {
        exclude group: 'com.android.support'
    }
 ```   

### 3）、使用 debugImplementation 来依赖仅在 debug 期间才会使用的库，如一些线下的性能检测工具。如下是一个示例代码：

```java
// 仅在debug包启用BlockCanary进行卡顿监控和提示的话，可以这么用
debugImplementation 'com.github.markzhai:blockcanary-android:1.5.0'
```


## 8、利用公司 Maven 仓库的本地缓存

当第一个开发引入了新库或者更新版本之后，公司的 Maven 仓库中就会缓存对应的库版本，通过这样的方式，其他开发同事就能够在项目构建时直接从公司的 Maven 仓库中拿到缓存。


## 9、Debug 构建时设置 minSdkVersion 为 21

这样，我们就可以避免因使用 MutliDex 而拖慢 build 速度。在主 Moudle 中的 build.gradle 中加入如下代码：

```java
    productFlavors {
        speed {
            minSdkVersion 21
        }
    }
```  
    
同步项目之后，我们在Android Studio右侧的 Build Variants 中选中 speedDebug 选项即可，如下图所示：


![](https://imgkr.cn-bj.ufileos.com/29089df7-738d-4ae1-bf0e-dfd08e8890ec.png)


需要注意的是，要注意我们当前项目的实际最低版本，比如它为 18，现在我们开启了 speedDebug，项目编写时就会以 21 为标准，此时，就 **需要注意 18 ~ 21 之间的 API**，例如我在布局中使用了 21 版本新出的 Material Design 的控件，此时就是没问题的，但实际我们需要对 21 版本以下的对应布局做相应的适配。

此外，我们也可以定义不同的 productFlavors，并且在 src 目录下新建对应的 flavor 名称标识的目录资源文件，以此实现在不同的渠道 APK 中采用不同的资源文件。


## 10、配置 gradle.properties 

通用的配置项如下所示：

```java
    // 构建初始化需要执行许多任务，例如java虚拟机的启动，加载虚拟机环境，加载class文件等等，配置此项可以开启线程守护，并且仅仅第一次编译时会开启线程（Gradle 3.0版本以后默认支持）
    org.gradle.daemon=true  
    
    // 配置编译时的虚拟机大小
    org.gradle.jvmargs=-Xmx2048m -XX:MaxPermSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8  
    
    // 开启并行编译，相当使用了多线程，仅仅适用于模块化项目（存在多个 Library 库工程依赖主工程）
    org.gradle.parallel=true  
    
    // 最大的优势在于帮助多 Moudle 的工程提速，在编译多个 Module 相互依赖的项目时，Gradle 会按需选择进行编译，即仅仅编译相关的 Module
    org.gradle.configureondemand=true   
    
    // 开启构建缓存，Gradle 3.5新的缓存机制，可以缓存所有任务的输出，
    // 不同于buildCache仅仅缓存dex的外部libs，它可以复用
    // 任何时候的构建缓存，设置包括其它分支的构建缓存
    org.gradle.caching=true
``` 
    
这里效果比较好一点的配置项就是 **配置编译时的虚拟机大小** 这项，我们来详细分析下其中参数的含义，如下所示：

- <span style="color:rgb(248,57,41);font-weight:bold;">-Xmx2048m：</span><span style="color:#0e88eb;font-weight:bold;">指定 JVM 最大允许分配的堆内存为 2048MB，它会采用按需分配的方式。</span>
- <span style="color:rgb(248,57,41);font-weight:bold;">-XX:MaxPermSize=512m：</span><span style="color:#0e88eb;font-weight:bold;">指定 JVM 最大允许分配的非堆内存为 512MB，同上堆内存一样也是按需分配的。</span>


## 11、配置 DexOptions

我们可以将 dexOptions 配置项中的 maxProcessCount 设定为 8，这样编译时并行的最大进程数数目就可以提升到 8 个。
    
    
## 12、使用 walle 提升打多渠道包的效率

walle 是 Android Signature V2 Scheme 签名下的新一代渠道包打包神器，它在 Apk 中的 APK Signature Block 区块添加了自定义的渠道信息以生成渠道包，因而提高了渠道包的生成效率。此外，它也可以作为单机工具来使用，也可以部署在 HTTP 服务器上来实时处理渠道包 Apk 的升级网络请求，有需要的同学可以参考美团的 [walle](https://github.com/Meituan-Dianping/walle)。


## 13、设置应用支持的语言

如果应用没有做国际化，我们可以让应用仅仅支持 **中文的资源配置**，即将 resConfigs 设置为 "zh"。如下所示：

```groovy
    android {
        defaultConfig {
            resConfigs "zh"
        }
    }
```


## 14、使用增量编译

Gradle 的构建方式通常来说细分为以下 <span style="color:#773098;font-weight:bold;">三种：</span>

- 1）、<span style="color:rgb(248,57,41);font-weight:bold;">Full Build：</span><span style="color:#0e88eb;font-weight:bold;">全量构建，即从0开始构建。</span><span style="color:orangered;font-weight:bold;"></span>
- 2）、<span style="color:rgb(248,57,41);font-weight:bold;">Incremental build java change：</span><span style="color:#0e88eb;font-weight:bold;">增量构建Java改变，修改源代码后的构建，且之前构建过。</span><span style="color:orangered;font-weight:bold;"></span>
- 3）、<span style="color:rgb(248,57,41);font-weight:bold;">Incremental build resource change：</span><span style="color:#0e88eb;font-weight:bold;">修改资源文件后的构建，且之前构建过。</span><span style="color:orangered;font-weight:bold;"></span>


在 Gradle 4.10 版本之后便默认使用了增量编译，它会测试自上次构建以来是否已更改任何 gradle task 任务输入或输出。如果还没有，Gradle 会将该任务认为是最新的，因此跳过执行其动作。由于 Gradle 可以将项目的依赖关系分析精确到类级别，因此，此时仅会重新编译受影响的类。如果在更老的版本需要启动增量编译，可以使用如下配置：

```groovy
    tasks.withType(JavaCompile) {
        options.incremental = true
    }
``` 
    
    
## 15、使用循环进行依赖优化（🔥）

在 Awesome-WanAndroid 项目的 app moudle 的 build.gradle 中，有将近几百行的依赖代码，如下所示：

```groovy
    dependencies {
        implementation fileTree(include: ['*.jar'], dir: 'libs')

        // 启动器
        api files('libs/launchstarter-release-1.0.0.aar')
        
         //base
        implementation rootProject.ext.dependencies["appcompat-v7"]
        implementation rootProject.ext.dependencies["cardview-v7"]
        implementation rootProject.ext.dependencies["design"]
        implementation rootProject.ext.dependencies["constraint-layout"]
        
        annotationProcessor rootProject.ext.dependencies["glide_compiler"]
        
         //canary
        debugImplementation (rootProject.ext.dependencies["leakcanary-android"]) {
            exclude group: 'com.android.support'
        }
        releaseImplementation (rootProject.ext.dependencies["leakcanary-android-no-op"]) {
            exclude group: 'com.android.support'
        }
        testImplementation (rootProject.ext.dependencies["leakcanary-android-no-op"]) {
            exclude group: 'com.android.support'
        }
        
        ...
```      

有没有一种好的方式不在 build.gradle 中写这么多的依赖配置？

有，就是 **使用循环遍历依赖**。答案似乎很简单，但是要想处理在依赖时遇到的所有情况，并不简单。下面，我直接给出相应的适配代码，大家可以直接使用。

首先，在 app 下的 build.gradle 的依赖配置如下所示：

```groovy
    // 处理所有的 aar 依赖
    apiFileDependencies.each { k, v -> api files(v)}

    // 处理所有的 xxximplementation 依赖
    implementationDependencies.each { k, v -> implementation v }
    debugImplementationDependencies.each { k, v -> debugImplementation v }
    releaseImplementationDependencies.each { k, v -> releaseImplementation v }
    androidTestImplementationDependencies.each { k, v -> androidTestImplementation v }
    testImplementationDependencies.each { k, v -> testImplementation v }
    debugApiDependencies.each { k, v -> debugApi v }
    releaseApiDependencies.each { k, v -> releaseApi v }
    compileOnlyDependencies.each { k, v -> compileOnly v }
    
    // 处理 annotationProcessor 依赖
    processors.each { k, v -> annotationProcessor v }
    
    // 处理所有包含 exclude 的依赖
    implementationExcludes.each { entry ->
        implementation(entry.key) {
            entry.value.each { childEntry ->
                exclude(group: childEntry)
            }
        }
    }
    debugImplementationExcludes.each { entry ->
        debugImplementation(entry.key) {
            entry.value.each { childEntry ->
                exclude(group: childEntry.key, module: childEntry.value)
            }
        }
    }
    releaseImplementationExcludes.each { entry ->
        releaseImplementation(entry.key) {
            entry.value.each { childEntry ->
                exclude(group: childEntry.key, module: childEntry.value)
            }
        }
    }
    testImplementationExclude.each { entry ->
        testImplementation(entry.key) {
            entry.value.each { childEntry ->
                exclude(group: childEntry.key, module: childEntry.value)
            }
        }
    }
    androidTestImplementationExcludes.each { entry ->
        androidTestImplementation(entry.key) {
            entry.value.each { childEntry ->
                exclude(group: childEntry.key, module: childEntry.value)
            }
        }
    }
 ```   
    
然后，在 config.gradle 全局依赖管理文件中配置好对应名称的依赖数组即可。代码如下所示：

```groovy
    dependencies = [
            // base
            "appcompat-v7"                      : "com.android.support:appcompat-v7:${version["supportLibraryVersion"]}"，
            ...
    ]
    
    annotationProcessor = [
            "glide_compiler"                    : "com.github.bumptech.glide:compiler:${version["glideVersion"]}",
            ...
    ]
    
    apiFileDependencies = [
            "launchstarter"                                   :"libs/launchstarter-release-1.0.0.aar"
    ]
    
    debugImplementationDependencies = [
            "MethodTraceMan"                                  : "com.github.zhengcx:MethodTraceMan:1.0.7"
    ]
    
    ...
    
    implementationExcludes = [
            "com.android.support.test.espresso:espresso-idling-resource:3.0.2" : [
                    'com.android.support' : 'support-annotations'
            ]
    ]
    
    ...
``` 
      
具体的代码示例可以在 Awesome-WanAndroid 的 [build.gradle](https://github.com/JsonChao/Awesome-WanAndroid/blob/master/app/build.gradle) 和 [config.gradle](https://github.com/JsonChao/Awesome-WanAndroid/blob/master/config.gradle) 上进行查看。


# 三、Gradle 常用命令

## 1、Gradle 查询命令

### 1）、查看主要任务

```gradle
    ./gradlew tasks
```
    
### 2）、查看所有任务，包括缓存任务等等

```gradle
    ./gradlew tasks --all
```   

## 2、Gradle 执行命令

### 1）、对某个module [moduleName]   的某个任务[TaskName] 运行

```gradle
    ./gradlew :moduleName:taskName
```

## 3、Gradle 快速构建命令

Gradle 提供了一系列的快速构建命令来替代 IDE 的可视化构建操作，如我们最常用的 clean、build 等等。需要注意的是，build 命令会把 debug、release 环境的包都构建出来。


### 1）、查看构建版本

```gradle
    ./gradlew -v
```   
    
### 2）、清除 build 文件夹

```gradle
    ./gradlew clean
```  
    
### 3）、检查依赖并编译打包

```gradle
    ./gradlew build
```  
    
### 4）、编译并安装 debug 包

```gradle
    ./gradlew installDebug
```
    
### 5）、编译并打印日志

```gradle
    ./gradlew build --info
```
    
### 6）、编译并输出性能报告，性能报告一般在构建工程根目录 build/reports/profile 下

```gradle
    ./gradlew build --profile
```
    
### 7）、调试模式构建并打印堆栈日志

```gradle
    ./gradlew build --info --debug --stacktrace
``` 
    
### 8）、强制更新最新依赖，清除构建后再构建
   
```gradle  
    ./gradlew clean build --refresh-dependencies
```   
    
### 9）、编译并打 Debug 包

```gradle
    ./gradlew assembleDebug
    # 简化版命令，取各个单词的首字母
    ./gradlew aD
``` 
    
### 10）、编译并打 Release 的包

```gradle
    ./gradlew assembleRelease
    # 简化版命令，取各个单词的首字母
    ./gradlew aR
```

## 4、Gradle 构建并安装命令

### 1）、Release 模式打包并安装

```gradle
    ./gradlew installRelease
```
    
### 2）、卸载 Release 模式包

```gradle
    ./gradlew uninstallRelease
``` 
    
### 3）、debug release 模式全部渠道打包

```gradle
    ./gradlew assemble
```   
    
## 5、Gradle 查看包依赖命令


### 1）、查看项目根目录下的依赖

```gradle
    ./gradlew dependencies
```  
    
### 2）、查看 app 模块下的依赖

```gradle
    ./gradlew app:dependencies
``` 
    
### 3）、查看 app 模块下包含 implementation 关键字的依赖项目

```gradle
    ./gradlew app:dependencies --configuration implementation
```   
    
# 四、使用 Build Scan 诊断应用的构建过程

在了解 Build Scan 之前，我们需要先来一起学习下旧时代的 Gradle build 诊断工具 Profile report。

## 1、Profile report

通常情况下，我们一般会使用如下命令来生成一份本地的构建分析报告：

```gradle    
    ./gradlew assembleDebug --profile
```  
    
这里，我们在 [Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid) App的根目录下运行这个命令，可以得到四块视图。下面，我们来了解下。


### 1）、Summary

**Gradle 构建信息的概览界面**，用于 **查看 Total Build Time、初始化（包含 Startup、Settings and BuildSrc、Loading Projects 三部分）、配置、任务执行的时间**。如下图所示：

![image](http://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/gradle_profile_report1.png?raw=true)


### 2）、Configuaration

**Gradle 配置各个工程所花费的时间**，我们可以看到 **All projects、app 模块以及其它模块单个的配置时间**。如下图所示：

![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/gradle_profile_report2.png?raw=true)


### 3）、Dependency Resolution

**Gradle 在对各个 task 进行依赖关系解析时所花费的时间**。如下图所示：


![image](https://raw.githubusercontent.com/JsonChao/Awesome-Android-Performance/master/screenshots/gradle_profile_report3png.png)


### 4）、Task Execution

**Gradle 在执行各个 Gradle task 所花费的时间**。如下图所示：


![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/gradle_profile_report4png.png?raw=true)


需要注意的是，Task Execution 的时间是所有 gradle task 执行时间的总和，实际上 **多模块的任务是并行执行的**。


## 2、Build Scan

**Build Scan 是官方推出的用于诊断应用构建过程的性能检测工具，它能分析出导致应用构建速度慢的一些问题**。在项目下使用如下命令即可开启 Build Scan 诊断：

```gradle    
    ./gradlew build --scan 
```  

如果你使用的是 Mac，使用上述命令时出现

```gradle
    zsh: permission denied: ./gradlew
``` 

可以加入下面的命给 gradlew 分配执行权限：

```gradle  
    chmod +x gradlew
```  

执行完 build --scan 命令之后，在命令的最后我们可以看到如下信息：

![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan3.png?raw=true)


可以看到，在 Publishing build scan 点击下面的链接就可以跳转到 Build Scan 的诊断页面。

需要注意的是，如果你是第一次使用 Build Scan，首先需要使用自己的邮箱激活 Build Scan。如下图界面所示：

![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan.png?raw=true)


这里，我输入了我的邮箱 chao.qu521@gmail.com，点击 Go！之后，我们就可以登录我们的邮箱去确认授权即可。如下图所示：

![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan2.png?raw=true)

直接点击 **Discover your build** 即可。

授权成功后，我们就可以看到 Build Scan 的诊断页面了。如下图所示：

![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan4.png?raw=true)


可以看到，在界面的右边有一系列的功能 tab 可供我们选择查看，这里默认是 Summary 总览界面，我们的目的是要查看 应用的构建性能，所以点击右侧的 Performance tab 即可看到如下图所示的构建分析界面：

![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan5.png?raw=true)


从上图可以看到，**Performance 界面中除了 Build、Configuration、Dependency resolution、Task execution 这四项外，还有 Daemon、Network activity、Settings and suggestions**。

在 Build 界面中，共有三个子项目，即 **Total build time、Total garbage collection time、Peak heap memory usage**，Total build time 里面的配置项前面我们已经分析过了，这里我们看看其余两项的含义，如下所示：

- <span style="color:rgb(248,57,41);font-weight:bold;">Total garbage collection time：</span><span style="color:#0e88eb;font-weight:bold;">总的垃圾回收时间。</span><span style="color:orangered;font-weight:bold;"></span>
- <span style="color:rgb(248,57,41);font-weight:bold;">Peak heap memory usage：</span><span style="color:#0e88eb;font-weight:bold;">最大堆内存使用。</span><span style="color:orangered;font-weight:bold;"></span>


对于 Peak heap memory usage 这一项来说，还有三个子项，其含义如下：

- 1）、<span style="color:rgb(248,57,41);font-weight:bold;">PS Eden Space：</span><span style="color:#0e88eb;font-weight:bold;">Young Generation 的 Eden（伊甸园）物理内存区域。程序中生成的大部分新的对象都在 Eden 区中。</span><span style="color:orangered;font-weight:bold;"></span>
- 2）、<span style="color:rgb(248,57,41);font-weight:bold;">PS Survivor Space：</span><span style="color:#0e88eb;font-weight:bold;">Young Generation 的 Eden 的 两个Survivor（幸存者）物理内存区域。当 Eden 区满时，还存活的对象将被复制到其中一个 Survivor 区，当此 Survivor 区满时，此区存活的对象又被复制到另一个 Survivor 区，当这个 Survivor 区也满时，会将其中存活的对象复制到年老代。</span><span style="color:orangered;font-weight:bold;"></span>
- 3）、<span style="color:rgb(248,57,41);font-weight:bold;">PS Old Gen：</span><span style="color:#0e88eb;font-weight:bold;">Old Generation，一般情况下，年老代中的对象生命周期都比较长。</span><span style="color:orangered;font-weight:bold;"></span>


由于我们的目的是关注项目的 build 时间，所以，我们直接关注到 Task execution 这一项。如下图所示：


![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan6.png?raw=true)


可以看到，Awesome-WanAndroid 项目中所有的 task 都是 Not cacheable 的。此时，我们往下滑动界面，可以看到所有 task 的构建时间。如下所示：


![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan7.png?raw=true)


如果，我们想查看一个 tinyPicPluginSpeedRelease 这一个 task 的执行详细，可以点击 :app:tinyPicPluginSpeedRelease 这一项，然后，就会跳转到 Timeline 界面，显示出 tinyPicPluginSpeedRelease 相应的执行信息。如下图所示：


![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan8.png?raw=true)


此外，这里我们点击弹出框右上方的第一个图标：Focus on task in timeline 即可看到该 task 在整个 Gradle build 时间线上的精确位置，如下图所示：

![image](https://github.com/JsonChao/Awesome-Android-Performance/blob/master/screenshots/build_scan9.png?raw=true)


至此，我们可以看到 Build Scan 的功能要比 Profile report 强大不少，所以我强烈建议优先使用它进行 Gradle 构建时间的诊断与优化。


# 五、总结

Gradle 每次构建的运行时间会随着项目编译次数越来少，因此为了准确评估 Gradle 构建提速的优化效果，我们可以在优化前后分别执行以下命令进行对比分析，如下所示：

```java
    gradlew --profile --recompile-scripts --offline --rerun-tasks assembleDebug
```

参数含义如下：

- <span style="color:rgb(248,57,41);font-weight:bold;">profile：</span><span style="color:#0e88eb;font-weight:bold;">开启性能检测。</span><span style="color:orangered;font-weight:bold;"></span>
- <span style="color:rgb(248,57,41);font-weight:bold;">recompile-scripts：</span><span style="color:#0e88eb;font-weight:bold;">不使用缓存，直接重新编译脚本。</span><span style="color:orangered;font-weight:bold;"></span>
- <span style="color:rgb(248,57,41);font-weight:bold;">offline：</span><span style="color:#0e88eb;font-weight:bold;">启用离线编译模式。</span><span style="color:orangered;font-weight:bold;"></span>
- <span style="color:rgb(248,57,41);font-weight:bold;">return-task：</span><span style="color:#0e88eb;font-weight:bold;">运行所有 gradle task 并忽略所有优化。</span><span style="color:orangered;font-weight:bold;"></span>


此外，Facebook 的 Buck 以及 Google 的 Bazel 都是优秀的编译工具，那么他们为什么没有使用开源的构建工具呢，主要有如下 <span style="color:#773098;font-weight:bold;">三点原因：</span>

- 1)、<span style="color:rgb(248,57,41);font-weight:bold;">统一编译工具：</span><span style="color:#0e88eb;font-weight:bold;">内部的所有项目都使用同一套构建工具，包括 Android、Java、iOS、Go、C++ 等。编译工具的统一优化会使所有项目受益。</span>
- 2)、<span style="color:rgb(248,57,41);font-weight:bold;">代码组织管理架构：</span><span style="color:#0e88eb;font-weight:bold;">Facebook 和 Google 的所有项目都放到同一个仓库里面，因此整个仓库非常庞大，并且，他们也不会使用 Git。目前 Google 使用的是[Piper](http://www.ruanyifeng.com/blog/2016/07/google-monolithic-source-repository.html)，Facebook 是基于[HG](https://www.mercurial-scm.org/)修改的，也是一种基于分布式的文件系统。</span><span style="color:orangered;font-weight:bold;"></span>
- 3)、<span style="color:rgb(248,57,41);font-weight:bold;">极致的性能追求：</span><span style="color:#0e88eb;font-weight:bold;">Buck 和 Bazel 的性能的确比 Gradle 更好，内部包含它们的各种编译优化。但是它们的定制型太强，而且对 Maven、JCenter 这样的外部依赖支持也不好。</span><span style="color:orangered;font-weight:bold;"></span>

但是，**[Buck](https://buck.build/) 和 [Bazel](https://github.com/bazelbuild/bazel) 编译构建工具内部的优化思路** 还是很值得我们学习和参考的，有兴趣的同学可以去研究下。下一篇文章，**我们将一起来学习 Gradle 中的必备基础 — groovy，这将会给我们后续的 Gradle 学习打下坚实的基础**，敬请期待。


## 参考链接：
---
1、[Gradle Github 地址](https://github.com/gradle/gradle)

2、[Gradle配置最佳实践](https://juejin.im/post/582d606767f3560063320b21#heading-25)

3、[提升 50% 的编译速度！阿里零售通 App 工程提效实践](https://juejin.im/post/5be3daf3e51d457844614d0b)

4、[Gradle 提速：每天为你省下一杯喝咖啡的时间](https://juejin.im/post/5be105fde51d455bad089fed)

5、[[大餐]加快gradle构建速度](http://halohoop.com/2017/06/13/meals-speedup_gradle_build/)

6、[Gradle模块化配置：让你的gradle代码控制在100行以内](https://www.jianshu.com/p/8d52afc1057d)

7、[Gradle Android-build 常用命令参数及解释](https://www.jianshu.com/p/a03f4f6ae31d)

8、[Android打包提速实践](https://www.jianshu.com/p/e456a5ac8613)

9、[GRADLE构建最佳实践](http://www.figotan.org/2016/04/01/gradle-on-android-best-practise/)

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

##  ●  微信群：

> **由于微信群已超过 200 人，麻烦大家想进微信群的朋友们，加我微信拉你进群。**
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。