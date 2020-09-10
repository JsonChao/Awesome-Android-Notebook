---

		title:  深入探索Android包体积优化
		date: 2020/2/18 20:21:00   
		tags: 
		- 性能优化
		categories: 性能优化
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的 [知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

在 **Android** 性能优化的知识体系当中，包体积优化一直被排在优先级比较低的位置，从而导致很多开发同学对自身应用的大小并不重视。在项目发展的历程中，一般可划分为如下三个阶段：


    初创期 => 成长期 => 成熟期
    
    
通常来说，**当应用处于成长期的中后阶段时，才会考虑去做系统的包体积优化**，因此，只有在这个阶段及之后，包体积优化带来的收益才是可观的。

那么，包体积优化能够给我们带来哪些 **收益** 呢？如何全面对应用的包体积进行 **系统分析** 及 **针对性优化** 呢？在这篇文章中，我们将一起进行深入地分析与探索。


# 思维导图大纲


![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abc9dab3bb8147448358c5057f52c130~tplv-k3u1fbpfcp-zoom-1.image)


# 目录

- 一、[瘦身优化及 Apk 分析方案](https://juejin.im/post/6844904103131234311#heading-4)
    - 1、瘦身优势
    - 2、APK 组成
    - 3、APK 分析
- 二、[代码瘦身方案探索](https://juejin.im/post/6844904103131234311#heading-21)
    - 1、Dex 探秘
    - 2、ProGuard
    - 3、D8 与 R8 优化
    - 4、去除 debug 信息与行号信息
    - 5、dex 分包优化
    - 6、使用 XZ Utils 进行 Dex 压缩
    - 7、三方库处理
    - 8、移除无用代码
    - 9、避免产生 Java access 方法
    - 10、利用 ByteX Gradle 插件平台中的代码优化插件
    - 11、小结
- 三、[资源瘦身方案探索](https://juejin.im/post/6844904103131234311#heading-64)
    - 1、冗余资源优化
    - 2、重复资源优化
    - 3、图片压缩
    - 4、使用针对性的图片格式
    - 5、资源混淆
    - 6、R Field 的内联优化
    - 7、资源合并方案
    - 8、资源文件最少化配置
    - 9、尽量每张图片只保留一份
    - 10、资源在线化
    - 11、统一应用风格
- 四、[So 瘦身方案探索](https://juejin.im/post/6844904103131234311#heading-88)
    - 1、So 移除方案
    - 2、So 移除方案优化版
    - 3、使用 XZ Utils 对 Native Library 进行压缩
    - 4、对 Native Library 进行合并
    - 5、删除 Native Library 中无用的导出 symbol
    - 6、So 动态下载
- 五、[其它优化方案](https://juejin.im/post/6844904103131234311#heading-95)
    - 1、插件化
    - 2、业务梳理
    - 3、转变开发模式
- 六、[包体积监控](https://juejin.im/post/6844904103131234311#heading-99)
    - 1、包体积监控的纬度
- 七、[瘦身优化常见问题](https://juejin.im/post/6844904103131234311#heading-101)
    - 1、怎么降低 Apk 包大小？
    - 2、Apk 瘦身如何实现长效治理？
- 八、[总结](https://juejin.im/post/6844904103131234311#heading-104)


下面，我们就先来了解下为什么要进行瘦身优化以及如何对 **Apk** 大小进行分析。


# 一、瘦身优化及 Apk 分析方案介绍

## 1、瘦身优势

我们首先来介绍下，**为什么我们需要做 APK 的瘦身优化？**

### APK 瘦身优化的原因

主要有 **三个方面** 的原因：

#### 1、下载转化率

**APK** 瘦身优化在实际的项目中优先级是比较低的，因为做了之后它的好处不是那么明显，尤其是那些还没有到 **稳定期** 的项目，我们都知道，**App** 的发展历程是从 **项目初期 => 成长期 => 稳定期**，对于处于 **发展初期与成长期** 的项目而言，可能会做 **启动优化、卡顿优化**，但是一般不会做 **瘦身优化**，**瘦身优化** 最主要的好处是对应用 **下载转化率** 的影响，**它是 App 业务运营的重要指标之一，在项目精细化运营的阶段是非常重要的**。因为如果你的 **App** 与其它同类型的 **App** 相比 **Apk** 体积要更小的话，那么你的 **App** 下载率就可能要高一些。而且，**包体积越小，用户下载等待的时间也会越短，所以下载转换成功率也就越高**。所以，安装包大小与下载转化率的关系 **大致是成反比** 的，即安装包越大，下载转换率就越小。一个 **80MB** 的应用，用户即使点了下载，也可能因为网络速度慢、突然反悔导致下载失败。而对于一个 **20MB** 的应用，用户点了下载之后，在犹豫要不要下的时候可能就已经下载完了。

而且，现在很多大型的 **App** 一般都会有一个 **Lite** 版本的 **App**，这个也是出于下载转化率方面的考虑。


#### 2、应用市场

**Google Play** 应用市场强制要求超过 **100MB** 的应用只能使用 **APK  扩展文件方式** 上传。当使用 **APK 扩展文件方式** 上传时，**Google Play** 会为我们的应用 **托管** 扩展文件，并将其 **免费提供** 给设备。**扩展文件将保存到设备的共享存储位置（SD 卡或可安装 USB 的分区；也称为“外部”存储），应用可以在其中访问它们**。在大多数设备上，**Google Play** 会在下载 **APK** 的同时下载扩展文件，因此应用在用户首次打开时便拥有了所需的一切。但是，在某些情况下，我们的应用必须在应用启动时从 **Google Play** 下载文件。**如果您想避免使用扩展文件，并且想要应用程序的下载大小大于100 MB，则应该使用  Android App Bundles 上传应用程序，此时应用程序最多可提供150 MB的压缩下载大小**。**Android App Bundles** 就是 **Android** 应用程序捆绑包，它能够让 **App** 以 **添加动态功能模块的方式** 去解决 **APK** 大小较大的问题。如下，就是由一个基本模块和两个动态功能模块组成的 **Android App Bundle APK** 的组成结构图：

    
![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cb15186e96e4f879f79c6d8bc005225~tplv-k3u1fbpfcp-zoom-1.image)


#### 3、渠道合作商的要求

此外，还有一个原因，当我们的 **App** 做大之后，可能需要跟各个手机厂商合作预装，这些 **渠道合作商会对你的 App 做详细的要求，只有达到相应的要求后才允许你的 App 预装到手机上**。而且，**越大的 App 其单价成本也会越高**。所以，瘦身也是我们项目做大之后一定会遇到的一个问题。


### 体积过大对 App 性能的影响

此外，包体积除了会影响 **应用的下载转化率** 之外，主要还会对 **App 三个方面** 的性能有一定的影响，如下所示：

- 1）、**安装时间**：比如 **文件拷贝、Library 解压，并且，在编译 ODEX 的时候，特别是对于 Android 5.0 和 6.0 系统来说，耗费的时间比较久，而 Android 7.0 之后有了 混合编译，所以还可以接受。最后，App 变大后，其 签名校验 的时间也会变长**。 
- 2）、**运行时内存：Resource 资源、Library 以及 Dex 类加载都会占用应用的一部分内存**。
- 3）、**ROM 空间**：如果应用的安装包大小为 **50MB**，那么启动解压之后很可能就已经超过 **100MB** 了。并且，如果 **闪存空间不足，很可能出现“写入放大”的情况**，它是闪存和固态硬盘（SSD）中一种不良的现象，闪存在可重新写入数据前必须先擦除，而擦除操作的粒度与写入操作相比低得多，执行这些操作就会多次移动（或改写）用户数据和元数据。因此，要改写数据，就需要读取闪存某些已使用的部分，更新它们，并写入到新的位置，如果新位置在之前已被使用过，还需连同先擦除；**由于闪存的这种工作方式，必须擦除改写的闪存部分比新数据实际需要的大得多。即最终可能导致实际写入的物理资料量是写入资料量的多倍**。


## 2、APK 组成

我们都知道，**Android** 项目最终会编译成一个 **.apk** 后缀的文件，实际上它就是一个 **压缩包**。因此，它内部还有很多不同类型的文件，这些文件，按照大小，共分为如下四类：

- 1）、**代码相关**：**classes.dex**，我们在项目中所编写的 **java** 文件，经过编译之后会生成一个 **.class** 文件，而这些所有的 **.class** 文件呢，它最终会经过 **dx** 工具编译生成一个 **classes.dex**。
- 2）、**资源相关**：**res**、**assets**、编译后的二进制资源文件 **resources.arsc** 和 清单文件 等等。**res** 和 **assets** 的不同在于 **res** 目录下的文件会在 **.R** 文件中生成对应的资源 **ID**，而 **assets** 不会自动生成对应的 **ID**，而是通过 **AssetManager** 类的接口来获取。此外，每当在 **res** 文件夹下放一个文件时，**aapt** 就会自动生成对应的 **id** 并保存在 **.R** 文件中，**但 .R 文件仅仅只是保证编译程序不会报错，实际上在应用运行时，系统会根据 ID 寻找对应的资源路径，而 resources.arsc 文件就是用来记录这些 ID 和 资源文件位置对应关系 的文件**。
- 3）、**So 相关**：**lib** 目录下的文件，这块文件的优化空间其实非常大。

此外，还有 **META-INF**，它存放了应用的 **签名信息**，其中主要有 **3个文件**，如下所示：

- 1）、**MANIFEST.MF**：其中每一个资源文件都有一个对应的 **SHA-256-Digest（SHA1)** 签名，**MANIFEST.MF** 文件的 **SHA256（SHA1）** 经过 **base64** 编码的结果即为 **CERT.SF** 中的 **SHA256（SHA1）-Digest-Manifest** 值。
- 2）、**CERT.SF**：除了开头处定义的 **SHA256（SHA1）-Digest-Manifest** 值，后面几项的值是对 **MANIFEST.MF** 文件中的每项再次 **SHA256（SHA1）** 经过 **base64** 编码后的值。
- 3）、**CERT.RSA：其中包含了公钥、加密算法等信息。首先，对前一步生成的 CERT.SF 使用了 SHA256（SHA1）生成了数字摘要并使用了 RSA 加密，接着，利用了开发者私钥进行签名。然后，在安装时使用公钥解密。最后，将其与未加密的摘要信息（MANIFEST.MF文件）进行对比，如果相符，则表明内容没有被修改。**


## 3、APK分析

下面，我们就来学习 **APK** 分析的 **四种常用方式**。

### 1、使用 ApkTool 反编译工具分析 APK

第一种方式，就是使用 **ApkTool** 这个反编译工具，它的官网地址如下：

[ApkTool 官方网站](https://github.com/iBotPeaches/Apktool)

其具体的 **反编译命令** 如下所示：

    
    apktool d xxx.apk
    
    
下面，我们就来使用 ApkTool 来对应用进行反编译。

#### ApkTool反编译实战

##### 1、下载并配置apktool

[apktool 下载配置官方文档](https://www.jianshu.com/go-wild?ac=2&url=https%3A%2F%2Fibotpeaches.github.io%2FApktool%2Finstall%2F)

我这里仅介绍 **Mac OS X** 平台上的下载配置，其它平台请点击上方链接查看。


- 1）、[下载脚本](https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/osx/apktool)，保存为 **apktool** 文件。
- 2）、[下载最新版 apktool.jar](https://bitbucket.org/iBotPeaches/apktool/downloads/)（fanqiang)
- 3）、将下载的 **jar** 包重命名为 **apktool.jar**。
- 4）、**配置环境变量**，这里有两种方案，如下所示：
    - 第一种是直接将 **apktool** 和 **apktool.jar** 移到 **/usr/local/bin** 目录，但是这里需要 **root** 权限，命令前加 **sudo**，回车后输入密码即可。
    - 第二种是在 **~/.bash_profile** 文件下配置，首先新建 **apktool** 文件夹，将两个文件放到这个文件下，打开终端，使用 **vim** 加上环境配置，其命令如下所示：
    
```
    // 1、使用vim命令在命令行打开.bash_profile文件，并可以在命令行
    // 上编辑，当然，你也可以直接打开.bash_profile文件
    vim ~/.bash_profile
    // 2、在.bash_profile最后加上这一行即可
    export PATH=前面路径/apktool:$PATH
    // 3、使编辑后的配置生效
    source ~/.bash_profile
```

- 5）、最后，使用以下命令将两个文件权限设置为 **可执行** 即可：


    sudo chmod a+x file


##### 2、使用ApkTool分析APK

我们在命令行下输入以下命令对 **APK** 进行反编译，如下所示：

    
    java -jar apktool_2.3.4.jar apktool d app-release.apk
    

反编译完成之后，它就会在当前的文件夹下面生成 **app-release** 的目录，目录结构如下所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd9982d1ddd94d7c96dbee90b8b379eb~tplv-k3u1fbpfcp-zoom-1.image)


这样我们就可以看到当前 **App的具体组成** 了。下面，我们介绍下第二种 **APK 分析** 的方式。


### 2、使用AS 2.2之后提供的Analyze APK

**Analyze APK** 具有如下功能：


- 1）、**可以直观地查看到 APK 的组成，比如大小、占比等等**。
- 2）、**查看 dex 文件的组成**。
- 3）、**对不同的 APK 进行对比分析**。


下面，我们就来具体实战一下，需要注意的是，我们可以 **直接将电脑上的 apk 拖进 AS 中就可以自动使用 Analyze APK 打开 apk**。然后，我们就可以看到 **APK 文件的绝对大小以及各组成文件的百分占比**，如下图所示：

    
![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5eafb86d5b8e4c59923fc6775b67d347~tplv-k3u1fbpfcp-zoom-1.image)


可以看到，[Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid)  应用的 **classes.dex** 的大小为 **3.3MB**，总占比为 **42.2%**。并且，**lib** 和 **res** 目录也有 **1.9MB**，总占比大概为 **25%**，因此，对于 **Awesome-WanAndroid App的优化方向就应该是 dex 为主、so 和 res 为辅** 了。此外，我们还可以查看 **classes.dex** 中还包含有哪些类，如下图所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc42a47d8a244cd4ab620739973999ae~tplv-k3u1fbpfcp-zoom-1.image)


我们平时在做 **竞品分析** 的时候，就能够很方便地来 **看一下我们 App 的竞品用到了哪些第三方 SDK**。同时，我们也可以从清单文件中很方便地查看 **APK** 文件的最终版本，因为 **Analyze APK** 能够直接对清单文件进行解析。

此外，在应用右上角还有一个 **Compare with previos APK** 的按钮，我们点击它之后，就可以 **将当前的 APK 与别的版本的 APK 进行对比**，这样就可以对新旧两个版本的 **APK** 文件大小进行对比。

接下来，我们再介绍下第三种 **APK** 分析的方式。


### 3、使用 nimbledroid 进行 APK 性能分析

[nimbledroid官网](https://nimbledroid.com)

nibledroid 是美国哥伦比亚大学的博士创业团队研发出来的分析 **Android App** 性能指标的系统，分析的方式有静态和动态两种方式，如下所示：


- 1）、**静态分析**：可以分析出APK安装包中**大文件排行榜，Dex 方法数和知名第三方 SDK 的方法数及占代码整体的比例**。
- 2）、**动态分析**：可以给出 **冷启动时间, 列出 Block UI 的具体方法, 内存占用, 以及 Hot Methods**, 从这些分析报告中, 可以 **定位出具体的优化点**。


它的使用方式其实非常简单，**只需要直接上传APK** 即可。然后，**nimbledroid** 网站的后台就会自动对 **APK** 进行分析，并最终给出一份 **全面的 APK 分析报告**。

下面，我们再来介绍最后一种 **APK** 分析工具，即二进制检查工具 **android-classshark**。


### 4、使用 android-classshark 进行 APK 分析

[android-classshark项目地址](https://github.com/google/android-classyshark)

**android-classshark** 是一个 **面向 Android 开发人员的独立二进制检查工具**，它可以 **浏览任何的 Android 可执行文件**，并且检查出信息，比如类的接口、成员变量等等，此外，它还可以支持多种格式，比如说 **APK、Jar、Class、So 以及所有的 Android 二进制文件如清单文件等等**。下面，我们就来使用 **android-classshark** 来进行一下实战。

#### android-classshark 实战

首先，我们从它的 **Github** 地址上下载对应的 **ClassyShark.jar**，地址如下所示：

[ClassyShark.jar-下载地址](https://github.com/google/android-classyshark/releases)

然后，我们双击打开 **ClassShark.jar**，**拖动我们的 APK 到它的工作空间即可**。接下来，我们就可以看到 **Apk** 的分析界面了，这里我们点击 **classes** 下的 **classes.dex**，在分析界面 **左边** 可以看到该 **dex 的方法数和文件大小**，并且，**最下面还显示出了该 dex 中包含有 Native Call 的类**。如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ba6c143123b41ee922e9f3a2362b719~tplv-k3u1fbpfcp-zoom-1.image)

此外，我们点击左上角的 **Methods count** 还可以切换到 **方法数环形图标统计界面**，我们不仅可以 **直观地看到各个包下的方法数和相对大小**，还可以看到**各个子包下的方法数和相对大小**。如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32b2ec4b420d450b9964a46dfa6e2d35~tplv-k3u1fbpfcp-zoom-1.image)


# 二、代码瘦身方案探索

在讲解如何对 **Dex** 进行优化之前，可能有很多同学对 **Dex** 还没有足够的了解，这里我们就先详细地了解下 **Dex**。


## 1、Dex 探秘

**Dex** 是 **Android** 系统的可执行文件，包含 **应用程序的全部操作指令以及运行时数据**。因为 **Dalvik** 是一种针对嵌入式设备而特殊设计的 **Java** 虚拟机，所以 **Dex 文件与标准的 Class 文件在结构设计上有着本质的区别**。当 **Java** 程序被编译成 **class** 文件之后，还需要使用 **dx** 工具将所有的 **class** 文件整合到一个 **dex** 文件中，这样 **dex 文件就将原来每个 class 文件中都有的共有信息合成了一体**，这样做的目的是 **保证其中的每个类都能够共享数据**，这在一定程度上 **降低了信息冗余**，同时也使得 **文件结构更加紧凑**。**与传统 jar 文件相比，Dex 文件的大小能够缩减 50% 左右**。关于 **Class** 文件与 **Dex** 文件的结果对比图如下所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8291c5f86a4f4584a5ddb31d0379f864~tplv-k3u1fbpfcp-zoom-1.image)


如果想深入地了解 **Dex** 文件格式，可以参见[Google 官方教程 - Dex格式](https://source.android.com/devices/tech/dalvik)。

**Dex** 一般在应用包体积中占据了不少比重，并且，**Dex** 数量越多，**App** 的安装时间也会越长。所以，优化它们可以说是 **重中之重**。下面，我们就来看看有哪些方式可以优化 **Dex** 这部分的体积。


## 2、ProGuard

**Java** 是一种跨平台的、解释型语言，而 **Java** 源代码被编译成 **中间 ”字节码”** 存储于 **Class** 文件之中。

### 那么，我们为什么要使用代码混淆呢？

由于跨平台的需要，**Java**   **字节码** 中包括了很多源代码信息，如变量名、方法名，并且通过这些名称来访问变量和方法，这些 **符号带有许多语义信息**，很 **容易被反编译成 Java 源代码**。为了防止这种现象，我们可以使用 Java 混淆器对 Java 字节码进行混淆。

代码混淆也被称为 **花指令**，它 **将计算机程序的代码转换成一种功能上等价，但是难以阅读和直接理解的形式**。混淆就是对发布出去的程序进行重新组织和处理，使得处理后的代码与处理前代码完成相同的功能，而混淆后的代码很难被反编译，即使反编译成功也很难得出程序的真正语义。混淆器的 **作用** 不仅仅是 **保护代码**，它也有 **精简编译后程序大小** 的作用，其 **通过缩短变量和函数名以及丢失部分无用信息等方式，能使得应用包体积减小**。


### 代码混淆的形式

目前，代码混淆的形式主要有 **三种**，如下所示：

- 1）、**将代码中的各个元素，比如类、函数、变量的名字改变成无意义的名字**。例如将 **hasValue** 转换成单个的字母 **a**。这样，反编译阅读的人就无法通过名字来猜测用途。
- 2）、**重写** 代码中的 **部分逻辑**，将它变成 **功能上等价**，但是又 **难以理解** 的形式。比如它会 **改变循环的指令、结构体**。
- 3）、**打乱代码的格式**，比如多加一些空格或删除空格，或者将一行代码写成多行，将多行代码改成一行。


### Proguard 的作用

在 **Android SDK** 里面集成了一个工具 — **Proguard**，它是一个免费的 **Java** 类文件 **压缩、优化、混淆、预先校验** 的工具。它的 **主要作用** 大概可以概括为 **两点**，如下所示：

- 1）、**瘦身：它可以检测并移除未使用到的类、方法、字段以及指令、冗余代码，并能够对字节码进行深度优化。最后，它还会将类中的字段、方法、类的名称改成简短无意义的名字**。
- 2）、**安全：增加代码被反编译的难度，一定程度上保证代码的安全**。


所以说，混淆不仅是保障 **Android 程序源码安全** 的 **第一道门槛**，而且在一定程度上，使用它能够减小 优化字节码 的大小。优化字节码 的处理流程如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af767dd730f54fefb631fadcde553e17~tplv-k3u1fbpfcp-zoom-1.image)

而它的作用具体可以细分三点，如下所示：

#### 1、压缩（Shrinking）

默认开启，以减小应用体积，移除未被使用的类和成员，并且 **会在优化动作执行之后再次执行**，因为优化后可能会再次暴露一些未被使用的类和成员。我们可以使用如下规则来关闭压缩：


    -dontshrink 关闭压缩
    

#### 2、优化（Optimization）

默认开启，在 **字节码级别执行优化**，让应用 **运行的更快**。使用如下规则可进行优化相关操作：

    
    -dontoptimize 关闭优化
    -optimizationpasses n 表示proguard对代码进行迭代优化的次数，Android一般为5


#### 3、混淆（Obfuscation）

默认开启，增大反编译难度，类和类成员会被随机命名，除非用 优化字节码 等规则进行保护。使用如下规则可以关闭混淆：

    
    -dontobfuscate 关闭混淆
    
    
### Proguard 的优化细节

**Proguard** 中所做的优化包括 **内联、修饰符、合并类和方法等 30 多种优化项，在特定的情况下，它尽可能地做了相应的优化**，下面列出了部分的 **优化细节**：

- **1)、优化了 Gson 库的使用**。
- **2)、把类都标记为 final**。
- **3)、把枚举类型简化为常量**。
- **4)、把一些类都垂直合并进当前类的结构中**。
- **5)、把一些类都水平合并进当前类的结构中**。
- **6)、移除 write-only 字段**。
- **7)、把类标记为私有的**。
- **8)、把字段的值跨方法地进行传递**。
- **9)、把一些方法标记为私有、静态或 final**。
- **10)、解除方法的 synchronized 标记**。
- **11)、移除没有使用的方法参数**。


### Proguard 的配置
    
混淆之后，默认会在工程目录 **app/build/outputs/mapping/release** 下生成一个 **mapping.txt** 文件，这就是 **混淆规则**，所以我们可以根据这个文件把混淆后的代码反推回原本的代码。要使用混淆，我们只需配置如下代码即可：


    buildTypes {
        release {
            // 1、是否进行混淆
            minifyEnabled true
            // 2、开启zipAlign可以让安装包中的资源按4字节对齐，这样可以减少应用在运行时的内存消耗
            zipAlignEnabled true
            // 3、移除无用的resource文件：当ProGuard 把部分无用代码移除的时候，
            // 这些代码所引用的资源也会被标记为无用资源，然后
            // 系统通过资源压缩功能将它们移除。
            // 需要注意的是目前资源压缩器目前不会移除values/文件夹中
            // 定义的资源（例如字符串、尺寸、样式和颜色）
            // 开启后，Android构建工具会通过ResourceUsageAnalyzer来检查
            // 哪些资源是无用的，当检查到无用的资源时会把该资源替换
            // 成预定义的版本。主要是针对.png、.9.png、.xml提供了
            // TINY_PNG、TINY_9PNG、TINY_XML这3个byte数组的预定义版本。
            // 资源压缩工具默认是采用安全压缩模式来运行，可以通过开启严格压缩模式来达到更好的瘦身效果。
            shrinkResources true
            // 4、混淆文件的位置，其中 proguard-android.txt 为sdk默认的混淆配置，
            // 它的位置位于android-sdk/tools/proguard/proguard-android.txt，
            // 此外，proguard-android-optimize.txt 也为sdk默认的混淆配置，
            // 但是它默认打开了优化开关。并且，我们可在配置混淆文件将android.util.Log置为无效代码，
            // 以去除apk中打印日志的代码。而 proguard-rules.pro 是该模块下的混淆配置。
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
        }
    }
    
    
首先，在注释1处，我们可以通过配置 **minifyEnabled** 来决定是否进行混淆。

然后，在注释2处，通过 **配置 zipAlignEnabled 为 true 可以让安装包中的资源按 4 字节对齐，这样可以减少应用在运行时的内存消耗**。

接着，在注释3处，配置 **shrinkResources** 为 **true** 可以移除无用的 **resource** 文件：**当 ProGuard 把部分无用代码移除的时候，这些代码所引用的资源也会被标记为无用资源，然后，系统会通过资源压缩功能将它们移除**。需要注意的是 **目前资源压缩器目前不会移除 values / 文件夹中定义的资源（例如字符串、尺寸、样式和颜色）**。开启后，**Android 构建工具会通过 ResourceUsageAnalyzer 来检查哪些资源是无用的，当检查到无用的资源时会把该资源替换成预定义的版本。主要是针对 .png、.9.png、.xml 提供了 TINY_PNG、TINY_9PNG、TINY_XML 这 3 个 byte 数组的预定义版本**。资源压缩工具默认是采用 **安全压缩模式** 来运行，可以通过开启 **严格压缩模式** 来达到 **更好的瘦身效果**。

最后，在注释
4处，我们可以配置混淆文件的位置，其中 **proguard-android.txt** 为 **sdk** 默认的混淆配置，它的位置位于 **android-sdk/tools/proguard/proguard-android.txt**，此外，**proguard-android-optimize.txt** 也是 **sdk** 默认的混淆配置，但是它 **默认打开了优化开关**。此外，我们也可以在配置混淆文件将 **android.util.Log** 置为无效代码，以去除 **apk** 中打印日志的代码。而 **proguard-rules.pro** 是该模块下的混淆配置。

在执行完 **ProGuard** 之后，ProGuard 都会在 `${project.buildDir}/outputs/mapping/${flavorDir}/` 生成以下文件：
  

文件名 | 描述
---|---
dump.txt |	APK中所有类文件的内部结构
mapping.txt	| 提供原始与混淆过的类、方法和字段名称之间的转换，可以通过proguard.obfuscate.MappingReader来解析
seeds.txt |	列出未进行混淆的类和成员
usage.txt |	列出从APK移除的代码


下面，我们再回顾下混淆的基本规则。


### 混淆的基本规则

    # * 表示仅保持该包下的类名，而子包下的类名还是会被混淆
    -keep class com.json.chao.wanandroid.*
    # ** 表示把本包和所含子包下的类名都保持
    -keep class com.json.chao.wanandroid.**
    
    # 既保持类名，又保持里面的内容不被混淆
    -keep class com.json.chao.wanandroid.* {*;}

    # 也可以使用Java的基本规则来保护特定类不被混淆，比如extend，implement等这些Java规则
    -keep public class * extends android.app.Activity
    
    # 保留MainPagerFragment内部类JavaScriptInterface中的所有public内容不被混淆
    -keepclassmembers class com.json.chao.wanandroid.ui.fragment.MainPagerFragment$JavaScriptInterface {
        public *;
    }
    
    # 仅希望保护类下的特定内容时需使用匹配符
    <init>;     //匹配所有构造器
    <fields>;   //匹配所有字段
    <methods>;  //匹配所有方法
    # 还可以在上述匹配符前面加上private 、public、native等来进一步指定不被混淆的内 容
    -keep class com.json.chao.wanandroid.app.WanAndroidApp {
        public <fields>;
    }
    # 也可以加入参数，以下表示用java.lang.String作为入参的构造函数不会被混淆
    -keep class com.json.chao.wanandroid.app.WanAndroidApp {
        public <init>(java.lang.String);
    }
    
    # 不需要保持类名，仅需要把该类下的特定成员保持不被混淆时使用keepclassmembers
    # 如果拥有某成员，要保留类和类成员使用-keepclasseswithmembers
    

了解完上面的这些混淆规则之后，相信我们已经能够根据我们当前的应用写出相应的混淆规则了。需要注意的是，**在 AndroidMainfest 中的类默认不会被混淆**，所以四大组件和 **Application** 的子类和 **Framework** 层下所有的类默认不会进行混淆，并且自定义的 **View** 默认也不会被混淆。因此，我们不需要手动在 **proguard-rules.pro** 中去添加如下代码：

    
    -keep public class * extends android.app.Activity
    -keep public class * extends android.app.Appliction
    -keep public class * extends android.app.Service
    -keep public class * extends android.content.BroadcastReceiver
    -keep public class * extends android.content.ContentProvider
    -keep public class * extends android.app.backup.BackupAgentHelper
    -keep public class * extends android.preference.Preference
    -keep public class * extends android.view.View
    -keep public class com.android.vending.licensing.ILicensingService
    

而 **Application** 和四大组件是必须在 **AndroidMainfest** 中进行注册的，所以如果想要通过 **混淆四大组件和 Application、自定义 View 的方式去减小APK的体积是行不通的**，因为没有规则去配置如何混淆四大组件和 **Application**。因此，**对于混淆的优化，我们能做的只能是
尽量保证 keep 范围的最小化，以此实现应用混淆程度的最大化**。在混淆配置中添加下列规则还可以在混淆之后输出最终的混淆配置：


    # 输出 ProGuard 的最终配置
    -printconfiguration configuration.txt


### 混淆实战

这里，我们就对 [Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid)  应用进行混淆，看看该应用混淆前后 **APK** 的体积变化。

下面这张图是 [Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid) 混淆前的 **APK** 组成结构图，可以看到占用了大概 **8.3MB** 的体积，其中 **dex** 部分占用了 **3.6MB**。


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3705f50506db47f2a6f1ed1d745bbe6b~tplv-k3u1fbpfcp-zoom-1.image)


混淆之后，**APK** 的体积会如何变化呢？我们看看 **混淆后的 APK 组成结构图**，如下所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d0d7dbc288c4981b8a56fdf88fadb72~tplv-k3u1fbpfcp-zoom-1.image)


可以看到，原先两个 **dex** 文件变为了一个，而且 **dex** 的大小也缩减到了 **2.2MB**，大小整整缩减了 **1.4MB**，**dex** 部分的压缩效果将近 **40%**。**APK** 整体的压缩效果也有 **17%**。所以，**混淆的确是 APK 瘦身的首选手段**。此外，**在 Android Studio 3.1 或之后的版本都会默认采用 D8 作为 Dex 的编译器**，并且，**在2019年10月，被认作为混淆的替代品的 R8 就已经默认集成进 Android Gradle plugin 中了**。下面，我们就看看 D8 与 R8 到底是如何优化 APK 的 dex 部分的。


## 3、D8 与 R8 优化

### D8 优化

#### 优化效果

D8 的 **优化效果** 总的来说可以归结为如下 **四点**：

- 1）、**Dex的编译时间更短**。
- 2）、**.dex文件更小**。
- 3）、**D8 编译的 .dex 文件拥有更好的运行时性能**。
- 4）、**包含 Java 8 语言支持的处理**。


#### 开启 D8

在 **Android Studio 3.0**
需要主动在 **gradle.properties** 文件中新增:

    android.enableD8 = true
    

**Android Studio 3.1** 或之后的版本 **D8** 将会被作为默认的 **Dex** 编译器。


### R8 优化

[R8 官方文档（目前已经开源）](https://r8.googlesource.com/r8)

**R8 是 Proguard 压缩与优化部分的替代品**，并且它仍然使用与 **Proguard** 一样的 **keep** 规则。如果我们仅仅想在 **Android Studio** 中使用 **R8**，当我们在 **build.gradle** 中打开混淆的时候，**R8** 就已经默认集成进 **Android Gradle plugin** 中了。

如果我们当前使用的是 Android Studio 3.4 或 Android Gradle 插件 3.4.0 及其更高版本，R8 会作为默认编译器。否则，我们 **必须要在 gradle.properties 中配置如下代码让 App 的混淆去支持 R8**，如下所示：


    android.enableR8=true
    android.enableR8.libraries=true


### 那么，R8 与混淆相比优势在哪里呢？

**ProGuard** 和 **R8** 都应用了基本名称混淆：它们 **都使用简短，无意义的名称重命名类，字段和方法**。他们还可以 **删除调试属性**。但是，**R8 在 inline 内联容器类中更有效，并且在删除未使用的类，字段和方法上则更具侵略性**。例如，**R8** 本身集成在 **ProGuard** V6.1.1 版本中，在压缩 **apk** 的大小方面，与 **ProGuard** 的 **8.5％** 相比，使用 **R8 apk** 尺寸减小了约 **10％**。并且，随着 **Kotlin** 现在成为 **Android** 的第一语言，**R8** 进行了 **ProGuard** 尚未提供的一些 **Kotlin** 的特定的优化。

从表面上看，**ProGuard** 和 **R8** 非常相似。它们都使用相同的配置，因此在它们之间进行切换很容易。放大来看的话，它们之间也存在一些差异。**R8 能更好地内联容器类，从而避免了对象分配**。但是 **ProGuard** 也有其自身的优势，具体有如下几点：

- 1）、**ProGuard 在将枚举类型简化为原始整数方面会更加强大**。它还传递常量方法参数，这通常对于使用应用程序的特定设置调用的通用库很有用。**ProGuard  的多次优化遍历通常可以产生一系列优化**。例如，第一遍可以传递一个常量方法参数，以便下一遍可以删除该参数并进一步传递该值。删除日志代码时，多次传递的效果尤其明显。**ProGuard 在删除所有跟踪（包括组成日志消息的字符串操作）方面更有效**。
- 2）、**ProGuard 中应用的模式匹配算法可以识别和替换短指令序列，从而提高代码效率并为更多优化打开了机会。在优化遍历的顺序中，尤其是数学运算和字符串运算可从中受益**。
- 3、最后，**ProGuard 具有独特的能力来优化使用 GSON 库将对象序列化或反序列化为 JSON 的代码**。该库严重依赖反射，这很方便，但效率低下。而 **ProGuard**  的优化功能可以 **通过更高效，直接的访问方式** 来代替它。


### R8 优化实战

接下来，我们就来看看 **Awesome-WanAndroid** 使用 **R8** 后，**APK** 体积的变化，如下图所示：

    
![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/85bec13a6b284817a7ae3e48998313fe~tplv-k3u1fbpfcp-zoom-1.image)


可以看到，相较于仅使用混淆后的 **APK** 而言，大小减少了 **0.1MB**，**Dex** 部分的优化效果大概为 **5%**，**APK** 整体的压缩效果也有 **1.5%** 左右。虽然从减少的 **APK** 大小来看，**0.1MB** 很少，但是比例并不小，如果你负责的是一个像微信、淘宝等规模的 **App**，它们的体积一般都将近 **100MB**，使用 **R8** 后也能减小 **1.5MB** 的大小。

此外，如果想单独对 **Dex 或 jar 包** 使用 **R8**，可以根据最上面的官方文档可以很快的在 **python** 环境下运行起来，其具体步骤如下所示：

#### 1、确保本地已经安装了python 2.7或更高版本。

#### 2、由于R8项目使用chromium项目提供的depot_tools管理依赖，因此先安装depot_tools。（下面仅介绍Mac版的安装）


- 1）、获取 depot_tools

    `git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git`


- 2）、获取 depot_tools 当前目录

    `pwd`


- 3）、添加环境变量
    - vim ~/.bash_profile：打开最后一行添加，如无此文件，可添加此文件
    - export PATH="$PATH:/PWD/depot_tools" : 其中 **PWD** 为刚才第二步获取的路径
    
    
- 4）、生效环境变量

    `source ~/.bash_profile`


#### 3、Downloading and building R8项目
    
    
    git clone https://r8.googlesource.com/r8
    cd r8
    tools/gradle.py d8 r8
    

#### 4、tools/gradle.py 脚本将会生成两个 jar 文件，即 build/libs/d8.jar  与 build/libs/r8.jar。

#### 5、下面的代码是使用 R8 在 out 目录下去生成优化后的 dex 文件：

    
    java -jar build/libs/r8.jar --release --output out --pg-conf proguard.cfg input.jar
    
    
D8 与 R8 的作用非常强大，而 Jake Wharton  大神最近一年多也在研究 D8 与 R8 的知识，如果想对 D8 与 R8 的实现细节有更多地了解，可以看看他的 [个人博客](https://jakewharton.com/)。

    
## 4、去除 debug 信息与行号信息

在讲解什么是 **deubg** 信息与行号信息之前，我们需要先了解 **Dex** 的一些知识。

我们都知道，**JVM** 运行时加载的是 **.class** 文件，而 **Android** 为了使包大小更加紧凑、运行时更加高效就发明了 **Dalvik** 和 **ART** 虚拟机，两种虚拟机运行的都是 **.dex** 文件，当然 **ART** 虚拟机还可以同时运行 oat 文件。

所以 **Dex** 文件里的信息内容和 **Class** 文件包含的信息是一样的，不同的是 **Dex** 文件对 **Class** 中的信息做了去重，一个 **Dex** 包含了很多的 **Class** 文件，并且在结构上有比较大的差异，**Class 是流式的结构，Dex  是分区结构，Dex 内部的各个区块间通过 offset 来进行索引**。

为了在应用出现问题时，我们能在调试的时候去显示相应的调试信息或者上报 **crash** 或者主动获取调用堆栈的时候能通过 **debugItem** 来获取对应的行号，我们都会在混淆配置中加上下面的规则：

    
    -keepattributes SourceFile, LineNumberTable
    

这样就会保留 **Dex** 中的 **debug 与行号信息**，此时的 **Dex 结构图** 如下所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ad2e4f7fbb24865be12537236b00b6f~tplv-k3u1fbpfcp-zoom-1.image)


从图中可以看到，**Dex** 文件的结构主要分为 **四大块：header 区，索引区，data 区，map 区**。而我们的 **debug** 与行号信息就保存在 **data 区中的 debugItems 区域**。而 **debug_items** 里面主要包含了 **两种信息**，如下所示：

- 1）、**调试的信息：包含函数的参数和所有的局部变量**。
- 2）、**排查问题的信息：包含所有的指令集行号与源文件行号的对应关系**。


根据 **Google** 官方的数据，**debugItem 一般占 Dex 的比例有 5% 左右**，如果我们能去除 **debug** 与行号信息，就能更进一步对 **Dex** 进行瘦身，但是会失去调试信息的功能，那么，**有什么方式可以去掉  debugItem，同时又能让 crash 上报的时候能拿到正确的行号呢？**

我们可以尝试直接修改 **Dex** 文件，**保留一小块  debugItem，让系统查找行号的时候指令集行号和源文件行号保持一致，这样任何监控上报的行号都直接变成了指令集行号**。

**每一个方法都会有一个 debugInfoItem，每一个 debuginfoItem 里面都有一个指令集行号和源文件行号的映射关系，这了我们直接把多余的 debugInfoItem 全部删掉，只保留了一个 debugInfoItem，这样所有的方法都会指向同一个 debugInfoItem**，并且这个 **debugInfoItem** 中的指令集行号和源文件行号保持一致，这样不管用什么方式来查找行号，拿到的都是指令集行号。需要注意的是，采用这种方案 **需要兼容所有虚拟机的查找方式**，因此 **仅仅保留一个 debugInfoItem 是不够的，需要对 debugInfoItem 进行分区，并且 debugInfoItem 表不能太大**。

关于如何去除 **Dex** 中的 **Debug** 信息是通过 **ReDex** 的 **StripDebugInfoPass** 来完成的，其配置如下所示：

    
    {
        "redex" : {
            "passes" : [
                "StripDebugInfoPass",
                "RegAllocPass"
            ]
        },
        "StripDebugInfoPass" : {
            "drop_all_dbg_info" : false,
            "drop_local_variables" : true,
            "drop_line_numbers" : false,
            "drop_src_files" : false,
            "use_whitelist" : false,
            "cls_whitelist" : [],
            "method_whitelist" : [],
            "drop_prologue_end" : true,
            "drop_epilogue_begin" : true,
            "drop_all_dbg_info_if_empty" : true
        },
        "RegAllocPass" : {
            "live_range_splitting": false
        }
    }
    
    
关于 **debuginfo** 的实战我们下面马上会开始，在此之前，我们先讲讲 **Dex** 分包中的另一个优化点。


## 5、Dex 分包优化

### Dex 分包优化原理

当我们的 **APK** 过大时，**Dex** 的方法数就会超过65536个，因此，必须采用 **mutildex** 进行分包，但是此时每一个 **Dex** 可能会调用到其它 **Dex** 中的方法，这种 **跨 Dex 调用的方式会造成许多冗余信息**，具体有如下两点：


- 1）、**多余的 method id：跨 Dex 调用会导致当前dex保留被调用dex中的方法id，这种冗余会导致每一个dex中可以存放的class变少，最终又会导致编译出来的dex数量增多，而dex数据的增加又会进一步加重这个问题**。
- 2)、**其它跨dex调用造成的信息冗余：除了需要多记录被调用的method id之外，还需多记录其所属类和当前方法的定义信息，这会造成 string_ids、type_ids、proto_ids 这几部分信息的冗余**。


为了减少跨 **Dex** 调用的情况，我们必须 **尽量将有调用关系的类和方法分配到同一个 Dex 中**。但是各个类相互之间的调用关系是非常复杂的，所以很难做到最优的情况。所幸的是，**ReDex** 的 [CrossDexDefMinimizer](https://github.com/facebook/redex/blob/master/opt/interdex/CrossDexRefMinimizer.cpp) 类分析了类之间的调用关系，并 **使用了贪心算法去计算局部的最优解（编译效果和dex优化效果之间的某一个平衡点）。使用 "InterDexPass" 配置项可以把互相引用的类尽量放在同个 Dex，增加类的 pre-verify，以此提升应用的冷启动速度**。

在 **ReDex** 中使用 **Dex** 分包优化跨 **dex** 调用造成的信息冗余的配置代码如下所示：

    
    
    {
        "redex" : {
            "passes" : [
                "InterDexPass",
                "RegAllocPass"
            ]
        },
        "InterDexPass" : {
            "minimize_cross_dex_refs": true,
            "minimize_cross_dex_refs_method_ref_weight": 100,
            "minimize_cross_dex_refs_field_ref_weight": 90,
            "minimize_cross_dex_refs_type_ref_weight": 100,
            "minimize_cross_dex_refs_string_ref_weight": 90
        },
        "RegAllocPass" : {
            "live_range_splitting": false
        },
        "string_sort_mode" : "class_order",
        "bytecode_sort_mode" : "class_order"
    }


为了衡量优化效果，我们可以使用 **Dex 信息有效率** 这个指标，公式如下所示：

    
    Dex 信息有效率 = define methods数量 / reference methods 数量
    

**如果 Dex 有效率在 80% 以上，就说明基本合格了**。


### 使用 ReDex 进行分包优化、去除 debug 信息及行号信息 

下面，我们就使用 [Redex](https://fbredex.com/docs/installation)  来对上一步生成的 **app-release-proguardwithr8.apk** 进行进一步的优化。（**macOS 环境下**）

#### 1、首先，我们需要输入一下命令去去安装 Xcode 命令行工具


    xcode-select --install


#### 2、然后，使用 homebrew 安装 redex 项目使用到的依赖库


    brew install autoconf automake libtool python3
    brew install boost jsoncpp
    

需要注意的是，2020年2月10号版本源码的 **redex** 需要的 **boost** 版本为 **V1.71** 及以上，当你使用 **brew install boost** 安装 **boost** 时可能获取到的 **boost** 版本会低于 **V1.71**，此时可能是 **brew** 版本需要更新，**使用 brew upgrade 去更新 brew 仓库的版本** 或者可以直接从 **boost** 官网下载最新的 [boost 源码](https://www.boost.org/) 至 **/usr/local/Cellar/** 目录下，我当前使用的是 [boost V1.7.2源码下载地址](https://dl.bintray.com/boostorg/release/1.72.0/source/) 中的 **boost_1_72_0.zip**。从 [深入探索 Android 启动优化](https://juejin.im/post/6844904093786308622) 时就提及到了 **Redex 的类重排优化**，当时卡在这一步，所以一直没法真正完成类的重排优化。
    
    
#### 3、接着，从 Github 上获取 ReDex 的源码并切换到 redex 目录下


    git clone https://github.com/facebook/redex.git
    cd redex
    
    
#### 4、下一步，使用 autoconf 和 make 去构建 ReDex


    # 如果你使用的是 gcc, 请使用 gcc-5
    autoreconf -ivf && ./configure && make -j4
    sudo make install
    
    
#### 5、然后，配置 Redex 的 config 代码

**在 Redex 在运行的时候，它是根据 redex/config/default.config 这个配置文件中的通道 passes 中添加不同的优化项来对 APK 的 Dex 进行处理的，我们可以参考 redex/config/default.config 这个默认的配置，里面的 passes 中不同的配置项都有特定的优化**。为了优化 **App** 的包体积，我们再加上 **interdex_stripdebuginfo.config** 中的配置项去删除 **debugInfo** 和减少跨 **Dex** 调用的情况，最终的 **interdex_stripdebuginfo.config 配置代码** 如下所示：


    {
        "redex" : {
            "passes" : [
                "StripDebugInfoPass",
                "InterDexPass",
                "RegAllocPass"
            ]
        },
        "StripDebugInfoPass" : {
            "drop_all_dbg_info" : false,
            "drop_local_variables" : true,
            "drop_line_numbers" : false,
            "drop_src_files" : false,
            "use_whitelist" : false,
            "cls_whitelist" : [],
            "method_whitelist" : [],
            "drop_prologue_end" : true,
            "drop_epilogue_begin" : true,
            "drop_all_dbg_info_if_empty" : true
        },
        "InterDexPass" : {
            "minimize_cross_dex_refs": true,
            "minimize_cross_dex_refs_method_ref_weight": 100,
            "minimize_cross_dex_refs_field_ref_weight": 90,
            "minimize_cross_dex_refs_type_ref_weight": 100,
            "minimize_cross_dex_refs_string_ref_weight": 90
        },
        "RegAllocPass" : {
            "live_range_splitting": false
        },
        "string_sort_mode" : "class_order",
        "bytecode_sort_mode" : "class_order"
    }
    
    
#### 6、最后，执行相应的 redex 优化命令 

这里我们使用 Redex 命令对上一 **Dex** 优化中得到的 **app_release-proguardwithr8.apk** 进行 **Dex 分包优化和去除 debugInfo**，它使用了贪心这种局部最优解的方式去减少跨 **Dex** 调用造成的信息冗余，命令如下所示（注意，**在 redex 的前面可能需要加上 Android sdk 的路径，因为 redex 中使用到了sdk下的zipalign工具**）：


    ANDROID_SDK=/Users/quchao/Library/Android/sdk redex --sign -s wan-android-key.jks -a wanandroid -p wanandroid -c ~/Desktop/interdex_stripdebuginfo.config -P app/proguard-rules.pro -o ~/Desktop/app-release-proguardwithr8-stripdebuginfo-interdex.apk ~/Desktop/app-release-proguardwithr8.apk


上述 **redex** 命令的 **关键参数含义** 如下所示：

- **--sign：对生成的apk进行签名**。
- **-s：配置应用的签名文件**。
- **-a: 配置应用签名的 key_alias**。
- **-p：配置应用签名的 key_password**。
- **-c：指定 redex 进行 Dex 处理时需要依据的 CONFIG 配置文件**。
- **-o：指定生成 APK 的全路径**。


使用上面的 redex 命令我们就可以对优化后的 **APK** 进行 **再签名和混淆**，等待一会后（如果你的 **APK** 的 **Dex** 数量和体积很大，可能会比较久），就会生成 **优化后的 APK：app-release-proguardwithr8-stripdebuginfo-interdex.apk**，如下图所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/866b76560c364429890b393f921ee068~tplv-k3u1fbpfcp-zoom-1.image)


可以看到，我们的 **APK** 大小几乎没有变化，这是因为当前的 **APK** 只有一个 **Dex**，并且 **第一个 Dex 默认不会优化**。为了能实际看到 **redex** 的优化效果，我们采用一个新项目来进行实验，项目地址如下所示：


[redex 优化 Apk 项目地址](https://github.com/AndroidAdvanceWithGeektime/Chapter22)


首先，引入一大堆开源库，尝试把 **Dex** 数量变多一些。然后直接通过 **assembleDebug** 编译即可。此外，为了可以更加清楚流程，我们可以在 **命令行输入 export TRACE=2 以便可以输出 redex 的日志**。最后，我们输入下面的 redex 命令删除 dex 中的 debugInfo 和减少跨 dex 调用的情况，如下所示：

    
    redex --sign -s ReDexSample/keystore/debug.keystore -a androiddebugkey -p android -c redex-test/interdex_stripdebuginfo.config -P ReDexSample/proguard-rules.pro  -o redex-test/strip_output.apk ReDexSample/build/outputs/apk/debug/ReDexSample-debug.apk


最终，我们看到前后的 APK 体积对比图如下所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/678fc286fb354096b39a3c6e32e2c1e6~tplv-k3u1fbpfcp-zoom-1.image)

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/480739e4997c4afd934c9c3d5812cd4f~tplv-k3u1fbpfcp-zoom-1.image)


可以看到，**APK 的大小从 14.2MB 减少到了 12.8MB，优化效果大概有10%**，效果还是比较明显的。此外，**如果你的 App 的 Dex 数量越多，那么优化的效果就会越大**。


## 6、使用 XZ Utils 进行 Dex 压缩

[XZ Utils 官文文档](https://tukaani.org/xz/)

XZ Utils 是具有高压缩率的免费通用数据压缩软件，它**同 7-Zip 一样，都是 LZMA Utils 的后继产品，内部使用了 LZMA/LZMA2 算法。LZMA 提供了高压缩比和快速解压缩，因此非常适合嵌入式应用**。**LZMA** 的 **主要功能** 如下：

- 1）、**压缩速度**：在3 GHz双核CPU上为3 MB / s。
- 2）、**减压速度**：在现代3 GHz CPU（Intel，AMD，ARM）上为20-50 MB / s。在简单的1 GHz RISC - CPU（ARM，MIPS，PowerPC）上为5-15 MB / s。
- 3）、**解压缩的较小内存要求**：8-32 KB + DictionarySize。
- 4）、**用于解压缩的代码大小**：2-8 KB（取决于速度优化）。


相对于典型的压缩文件而言，**XZ Utils 的输出比 gzip 小 30％，比 bzip2 小 15％**。在 **FaceBook** 的 **App** 中就使用了 **Dex 压缩** 的方式，而且它 **将 Dex 压缩后的文件都放在了 assets 目录中**，如下图所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c254d22ef56a4ef887eda90c85e1644f~tplv-k3u1fbpfcp-zoom-1.image)

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c82278955c94710a5e53168d9613a94~tplv-k3u1fbpfcp-zoom-1.image)


我们先看到上图中的 **classes.dex，其中仅包含了启动时要用到的类，这样可以为 Dex 压缩文件 secondary.dex.jar.xzs 的解压争取时间**。

此外，在 secondary.dex.jar.xzs 文件的下面，我们注意到，**有一系列的 secondary-x.dex.jar.xzs.tmp~.meta 文件，它保存了压缩前每一个 Dex 文件的映射元数据信息，在应用首次启动解压的时候我们还需要用到它**。

尽管 **classes.dex 为首次启动解压 Dex 压缩文件争取了时间，但是由于文件太大，在低端机上的解压时间可能会有 3~5s**。

而且，**当 Dex  非常多的时候会增加应用的安装时间，如果还使用了压缩 Dex 的方式，那么首次生成 ODEX 的时间可能就会超过1分钟**。为了解决这个问题，**Facebook** 使用了 **oatmeal** 这套工具去 **根据 ODEX 文件的格式，自己生成了一个 ODEX 文件**。而在 **正常的流程** 下，**系统会使用 fork 子进程的方式去处理 dex2oat 的过程**。

但是，**oatmeal** 采用了 **代理 dex2oat 省去 fork 进程所带来耗时** 的这种方式，**如果在1个 10MB 的 Dex，可以将 dex2oat 的耗时降至 100ms，而在 Android 5.0 上生成一个 ODEX 的耗时大约在 10 秒以上，在 Android 8.0 使用 speed 模式也需要 1 秒左右的时间**。但是由于 **每个 Android 系统版本的 ODEX 格式都有一些差异，oatmeal  需要分版本适配**，因此 Dex 压缩的方案我们可以先压压箱底。


## 7、三方库处理

实际的开发过程中，我们会用到各种各样的三方库。尤其当项目变大之后，开发人员众多，因此引入的三方库也会非常多，比如说，有人引入了一个 **Fresco** 图片库，然后这个库你可能不熟悉，你会引入一个 **Glide**，并且另一个人它可能又会引入他熟悉的图片库 **Picasso**，所以项目中可能会存在多个相同功能的三方 **SDK**，这一点，在大型项目当中一定会存在。因此，我们在做代码瘦身的时候，需要将三方库进行统一，比如说 **将图片加载库、网络库、数据库以及其他基础库进行统一，去掉冗余的库**。

同时，在选择第三方 **SDK** 的时候，我们可以将包大小作为选择的指标之一，我们应该 **尽可能地选择那些比较小的库来实现相同的功能**。例如，对于图片加载功能来说，**Picasso、Glide、Fresco** 它们都可以实现，但是你引入 **Fresco** 之后会导致包大小增加很多，而 **Picasso** 却只增加了不到 **100kb**，所以引入不同的三方 **SDK** 对包大小的影响是不一样的。这里，我们可以使用 **AS 插件 Android Methods Count**，安装之后，它会**自动在 build.gradle 文件中显示你引入的三方库的方法数**。

最后，如果我们引入三方库的时候，可以 **只引入部分需要的代码**，而不是将整个包的代码都引入进来。很多库的代码结构都设计的比较好，比如 **Fresco**，它将图片加载的各个功能，如 **webp、gif 功能进行了剥离，它们都处于单个的库当中**。如果我们只需要 **Fresco** 的 **webp** 功能，那我们可以将除 **webp** 之外的别的库都给删掉，这样你引入的三方库就很小了，包大小就降下来了。如下所图所示，我们可以**仅仅保留 Fresco 的 webp 功能**，其它依赖都可以去掉。

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd139fbeec854321b1e0f84aeb4c03f5~tplv-k3u1fbpfcp-zoom-1.image)


如果你引入的三方库 **没有进行过结构剥离**，就需要 **修改源码，只提取出来你需要的功能即可**。


## 8、移除无用代码

移除无用代码时我们经常会碰到下面两个问题：

- 1）、**业务代码只增不减**。
- 2）、**代码太多不敢删除**。


这里，有一个很好的方法可以 **准确地判断哪些类在线上环境下用户肯定不会用到了**。我们可以通过 **AOP** 的方式来做，对于 **Activity** 来说，其实非常简单，我们只需要 **在每个 Activity 的 onCreate 当中加上统计** 即可，然后**到了线上之后，如果这个 Activity 被统计了，就说明它还在被使用**。而对于那些 **不是 Activity 的类**，我们可以 **利用 AOP 来切它们的构造函数**，一个类如果它被使用，那它的构造函数肯定会被调用到。例如，下面就是 **使用 AspectJ 对某个包下的类进行构造函数切面** 的代码：


    @After("execution(org.jay.launchstarter.Task.new(..)")
    public void newObject(JoinPoint point) {
        LogHelper.i(" new " + point.getTarget().getClass().getSimpleName());
    }
    

其中，**new** 表示是 **切的构造函数**，括号中的 .. 表示的是 **匹配所有构造参数**。此外，我们也可以直接使用 [coverage 插件](https://github.com/bytedance/ByteX/blob/master/coverage/README-zh.md) 来做 **线上无用代码分析**，需要注意的是，**在注册上报数据的时候记得把服务器名改为自己的**。

最后，我们也可以在线下使用 [Simian工具](https://blog.csdn.net/Love667767/article/details/53558382) 或者 Lint 来 **扫描出重复的代码**。

### 使用 Lint 检测无效代码

```java
步骤：点击菜单栏 Analyze -> Run Inspection by Name -> unused declaration -> Moudule ‘app’ -> OK
```


## 9、避免产生 Java access 方法

### access 方法是什么？

为了能提供内部类和其外部类直接访问对方的私有成员的能力，又不违反封装性要求，**Java 编译器在编译过程中自动生成 package 可见性的静态 access$xxx 方法，并且在需要访问对方私有成员的地方改为调用对应的 access 方法**。


### 避免产生 access 方法的方式

主要有 **两种方式** 避免产生 **access** 方法：

- 1）、**在开发过程中需要注意在可能产生 access 方法的情况下适当调整，比如去掉 private，改为 package 可见性**。
- 2）、**使用 ASM 在编译时删除生成的 access 方法**。


因为优化效果不是很明显，这里就不多介绍了，具体的实现细节可参见 [西瓜视频 apk 瘦身之 Java access 方法删除](https://mp.weixin.qq.com/s/ZHisCVjO_ZrtvvEWBYUQFQ)，此外，在 **ReDex** 中也提供了  [access-marking](https://github.com/facebook/redex/tree/master/opt/access-marking) 这个功能去除代码中的 **Access** 方法，并且，在 ReDex 还有 [type-erasure](https://github.com/facebook/redex/tree/master/opt/type-erasure) 的功能，它 **与 access-marking 的优化效果一样，不仅能减少包大小，也能提升 App 的启动速度**。


## 10、利用 ByteX Gradle 插件平台中的代码优化插件

如果你想在项目的编译阶段去除 **access** 方法，这里我更加建议直接使用 **ByteX** 的 [access_inline](https://github.com/bytedance/ByteX/blob/master/access-inline-plugin/README-zh.md) 插件。除了 **access_inlie** 之外，在 **ByteX** 中还有 **四个** 很实用的代码优化 **Gradle** 插件可以帮助我们有效减小 **Dex** 文件的大小，如下所示：

- 1、编译期间 **内联常量字段**：[const_inline](https://github.com/bytedance/ByteX/blob/master/const-inline-plugin/README-zh.md)。
- 2、编译期间 **移除多余赋值代码**：[field_assign_opt](https://github.com/bytedance/ByteX/blob/master/field-assign-opt-plugin/README-zh.md)。
- 3、编译期间 **移除 Log 代码**：[method_call_opt](https://github.com/bytedance/ByteX/blob/master/method-call-opt-plugin/README-zh.md)。
- 4、编译期间 **内联 Get / Set 方法**：[getter-setter-inline-plugin](https://github.com/bytedance/ByteX/blob/master/getter-setter-inline-plugin/README-zh.md)。


## 11、小结

回顾下我们上述使用的各种 **Dex** 优化方式，其中，不少优化项都使用到了 **ReDex**。对于 ReDex 来说，目前它提供的比较强大的功能有 **五种**，分别如下所示：

- 1）、**Interdex：类重排和文件重排、Dex 分包优化。其中对于类重排和文件重排，Google 在 Android 8.0 的时候引入了 Dexlayout，它是一个用于分析 dex 文件，并根据配置文件对其进行重新排序的库。与 ReDex 类似，Dexlayout 通过将经常一起访问的部分 dex  文件集中在一起，程序可以因改进文件位置从而拥有更好的内存访问模式，以节省 RAM  并缩短启动时间。不同于ReDex的是它使用了运行时配置信息对 Dex  文件的各个部分进行重新排序。因此，只有在应用运行之后，并在系统空闲维护的时候才会将 dexlayout 集成到 dex2oat 的设备进行编译**。
- 2）、**Oatmeal：直接生成 Odex 文件**。
- 3）、**StripDebugInfo：去除 Dex 中的 Debug 信息**。
- 4）、**源码中 access-marking 模块：删除 Java access 方法**。
- 5）、**源码中 type-erasure 模块：类型擦除**。

可以看到，**ReDex** 的功能非常强大，**如果能够深入了解 ReDex 源码中的各个功能模块的实现，你将具有非常强硬的技术资本**。

最近，**抖音 Android 团队** 已经将上述部分模块的实现以 **Gradle Transform + ASM** 的形式集成进了  [ByteX](https://github.com/bytedance/ByteX)，建议掌握其实现原理后，大家可以直接在这个字节码插件开发平台上开发自己的 **Gradle** 插件。

最后，还有一些 **代码编写方面的优化**，如可以在开发过程 **尽量减少 enum 的使用，每减少一个 enum 可以减少大约 1.0 到 1.4 KB 的大小**。


# 三、资源瘦身方案探索

众所周知，**Android** 构建工具链中使用了 **AAPT/AAPT2** 工具来对资源进行处理，**Manifest、Resources、Assets 的资源经过相应的 ManifesMerger、ResourcesMerger、AssetsMerger 资源合并器将多个不同 moudule 的资源合并为了 MergedManifest、MergedResources、MergedAssets。然后，它们被 AAPT 处理后生成了 R.java、Proguard Configuration、Compiled Resources**。如下图左上方所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb7c62a9cc574a9fa117d08b6b53d96c~tplv-k3u1fbpfcp-zoom-1.image)

其中 **Proguard Configuration、Compiled Resources** 的 **作用** 如下所示：

- **Proguard Configuration：这是AAPT工具为Manifest中声明的四大组件与布局文件中使用的各种Views所生成的混淆配置**，该文件通常存放在 `${project.buildDir}/${AndroidProject.FD_INTERMEDIATES}/proguard-rules/${flavorName}/${buildType}/aapt_rules.txt`。
- **Compiled Resources：它是一个Zip格式的文件**，这个文件的路径通常为 `${project.buildDir}/${AndroidProject.FD_INTERMEDIATES}/res/resources-${flavorName}-${buildType}-stripped.ap_`。在经过 **zip** 解压之后，可以发现它 **包含了res、AndroidManifest.xml和resources.arsc 这三部分**。并且，从上面的 **APK** 构建流程中可以得知，**Compiled Resources** 会被 **apkbuilder** 打包到 **APK** 包中，它其实就是 **APK** 的**资源包**。因此，我们可以 **通过 Compiled Resources 文件来修改不同后缀文件资源的压缩方式来达到瘦身效果的**。但是需要注意的是，**resources.arsc 文件最好不要压缩存储，如果压缩会影响一定的性能，尤其是在冷启动时间方面造成的影响**。并且，**如果在 Android 6.0 上开启了 android:extractNativeLibs=”false” 的话，So 文件也不能被压缩**。


## 1、冗余资源优化

### 1、使用 Lint 的 Remove Unused Resource

**APK** 的资源主要包括图片、**XML**，与冗余代码一样，它也可能遗留了很多旧版本当中使用而新版本中不使用的资源，这点在快速开发的 **App** 中更可能出现。我们可以通过点击右键，选中 **Refactor**，然后点击 **Remove Unused Resource => preview** 可以预览找到的无用资源，点击 **Do Refactor** 可以去除冗余资源。如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d04c3f5318dc44bf9b241a353048d497~tplv-k3u1fbpfcp-zoom-1.image)


需要注意的，**Android Lint 不会分析 assets 文件夹下的资源，因为 assets 文件可以通过文件名直接访问，不需要通过具体的引用，Lint 无法判断资源是否被用到**。


### 2、优化 shrinkResources 流程真正去除无用资源

**resources.arsc** 中可能会存在很多 **无用的资源映射**，我们可以使用 [android-arscblamer](https://github.com/google/android-arscblamer)，它是一个命令行工具，能够 **解析 resources.arsc  文件并检查出可以优化的部分**，比如一些空的引用。

此外，当我们通过 **shrinkResources true** 来 **开启资源压缩**，资源压缩工具只会把无用的资源替换成预定义的版本而不是移除。那么，**如何高效地对无用资源自动进行去除呢？**

我们可以 **在 Android 构建工具执行 package${flavorName}Task 之前通过修改 Compiled Resources 来实现自动去除无用资源**，具体的实现原理如下：

#### 1）、首先，收集 Compiled Resources 中被替换的预定义版本的资源名称

**通过查看 Zip 格式资源包中每个 ZipEntry 的 CRC-32 checksum 来寻找被替换的预定义资源，预定义资源的 CRC-32 定义在 ResourceUsageAnalyze 中**，如下所示：

    
    // A 1x1 pixel PNG of type BufferedImage.TYPE_BYTE_GRAY
    public static final long TINY_PNG_CRC = 0x88b2a3b0L;

    // A 3x3 pixel PNG of type BufferedImage.TYPE_INT_ARGB with 9-patch markers
    public static final long TINY_9PNG_CRC = 0x1148f987L;

    // The XML document <x/> as binary-packed with AAPT
    public static final long TINY_XML_CRC = 0xd7e65643L;


#### 2）、然后，使用 [android-chunk-utils](https://github.com/madisp/android-chunk-utils)  把 resources.arsc 中对应的定义移除。

#### 3）、最后，删除资源包中对应的资源文件即可。


## 2、重复资源优化

在大型 **App** 项目的开发中，一个 **App** 一般会有多个业务团队进行开发，其中每个业务团队在资源提交时的资源名称可能会有重复的，这将会 **引发资源覆盖的问题**，因此，每个业务团队都会为自己的 **资源文件名添加前缀**。这样就导致了这些资源文件虽然 **内容相同**，但因为 **名称的不同而不能被覆盖**，最终都会被集成到 **APK** 包中。这里，我们还是可以 **在 Android 构建工具执行 package${flavorName}Task 之前通过修改 Compiled Resources 来实现重复资源的去除**，具体放入实现原理可细分为如下三个步骤：

- 1）、首先，**通过资源包中的每个ZipEntry的CRC-32 checksum来筛选出重复的资源**。
- 2）、然后，**通过[android-chunk-utils](https://github.com/madisp/android-chunk-utils)修改resources.arsc，把这些重复的资源都重定向到同一个文件上**。
- 3）、最后，**把其它重复的资源文件从资源包中删除，仅保留第一份资源**。


具体的实现代码如下所示：

    variantData.outputs.each {
        def apFile = it.packageAndroidArtifactTask.getResourceFile();

        it.packageAndroidArtifactTask.doFirst {
            def arscFile = new File(apFile.parentFile, "resources.arsc");
            JarUtil.extractZipEntry(apFile, "resources.arsc", arscFile);

            def HashMap<String, ArrayList<DuplicatedEntry>> duplicatedResources = findDuplicatedResources(apFile);

            removeZipEntry(apFile, "resources.arsc");

            if (arscFile.exists()) {
                FileInputStream arscStream = null;
                ResourceFile resourceFile = null;
                try {
                    arscStream = new FileInputStream(arscFile);

                    resourceFile = ResourceFile.fromInputStream(arscStream);
                    List<Chunk> chunks = resourceFile.getChunks();

                    HashMap<String, String> toBeReplacedResourceMap = new HashMap<String, String>(1024);

                    // 处理arsc并删除重复资源
                    Iterator<Map.Entry<String, ArrayList<DuplicatedEntry>>> iterator = duplicatedResources.entrySet().iterator();
                    while (iterator.hasNext()) {
                        Map.Entry<String, ArrayList<DuplicatedEntry>> duplicatedEntry = iterator.next();

                        // 保留第一个资源，其他资源删除掉
                        for (def index = 1; index < duplicatedEntry.value.size(); ++index) {
                            removeZipEntry(apFile, duplicatedEntry.value.get(index).name);

                            toBeReplacedResourceMap.put(duplicatedEntry.value.get(index).name, duplicatedEntry.value.get(0).name);
                        }
                    }

                    for (def index = 0; index < chunks.size(); ++index) {
                        Chunk chunk = chunks.get(index);
                        if (chunk instanceof ResourceTableChunk) {
                            ResourceTableChunk resourceTableChunk = (ResourceTableChunk) chunk;
                            StringPoolChunk stringPoolChunk = resourceTableChunk.getStringPool();
                            for (def i = 0; i < stringPoolChunk.stringCount; ++i) {
                                def key = stringPoolChunk.getString(i);
                                if (toBeReplacedResourceMap.containsKey(key)) {
                                    stringPoolChunk.setString(i, toBeReplacedResourceMap.get(key));
                                }
                            }
                        }
                    }

                } catch (IOException ignore) {
                } catch (FileNotFoundException ignore) {
                } finally {
                    if (arscStream != null) {
                        IOUtils.closeQuietly(arscStream);
                    }

                    arscFile.delete();
                    arscFile << resourceFile.toByteArray();

                    addZipEntry(apFile, arscFile);
                }
            }
        }
    }
    

然后，我们再看看图片压缩这一项。


## 3、图片压缩

一般来说，**1000行代码在APK中才会占用 5kb 的空间**，而图片呢，一般都有 **100kb** 左右，所以说，对图片做压缩，它的收益明显是更大的，而往往处于快速开发的 **App** 没有相关的开发规范，**UI** 设计师或开发同学如果忘记了添加图片时进行压缩，添加的就是原图，那么包体积肯定会增大很多。对于图片压缩，我们可以在 [tinypng](https://tingpng.com/) 这个网站进行图片压缩，但是如果 **App** 的图片过多，一个个压缩也是很麻烦的。因此，我们可以 **使用 [McImage](https://github.com/smallSohoSolo/McImage/blob/master/README-CN.md)、[TinyPngPlugin](https://github.com/Deemonser/TinyPngPlugin) 或 [TinyPIC_Gradle_Plugin](https://github.com/meili/TinyPIC_Gradle_Plugin/blob/master/README.zh-cn.md) 来对图片进行自动化批量压缩**。但是，需要注意的是，**在 Android 的构建流程中，AAPT 会使用内置的压缩算法来优化 res/drawable/ 目录下的 PNG 图片，但这可能会导致本来已经优化过的图片体积变大**，因此，可以通过在 **build.gradle** 中 **设置 cruncherEnabled 来禁止 AAPT 来优化 PNG 图片**，代码如下所示：


    aaptOptions {
        cruncherEnabled = false
    }
    

此外，我们还要注意对图片格式的选择，对于我们普遍使用更多的 **png** 或者是 **jpg** 格式来说，相同的图片转换为 **webp** 格式之后会有大幅度的压缩。**对于 png 来说，它是一个无损格式，而 jpg 是有损格式。jpg 在处理颜色图片很多时候根据压缩率的不同，它有时候会去掉我们肉眼识别差距比较小的颜色，但是 png 会严格地保留所有的色彩**。所以说，在图片尺寸大，或者是色彩鲜艳的时候，**png** 的体积会明显地大于 **jpg**。

下面，我们就着重讲解下如何针对性地选择图片格式。


## 4、使用针对性的图片格式

在 **Google I/O 2016** 中，讲到了如何选择相应的图片格式。首先，**如果能用 VectorDrawable 来表示的话，则优先使用 VectorDrawable；否则，看是否支持 WebP，支持则优先用 WebP；如果也不能使用 WebP，则优先使用 PNG，而 PNG 主要用在展示透明或者简单的图片，对于其它场景可以使用 JPG 格式**。简单来说可以归结为如下套路：


    VD（纯色icon）->WebP（非纯色icon）->Png（更好效果） ->jpg（若无alpha通道）


用 **图形化** 的形式如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aed55a72bcb44f949264249d738168f2~tplv-k3u1fbpfcp-zoom-1.image)


使用矢量图片之后，它能够有效的减少应用中图片所占用的大小，**矢量图形在 Android 中表示为 VectorDrawable 对象**。它 **仅仅需100字节的文件即可以生成屏幕大小的清晰图像**，但是，**Android 系统渲染每个 VectorDrawable 对象需要大量的时间，而较大的图像需要更长的时间**。 因此，建议 **只有在显示纯色小 icon 时才考虑使用矢量图形**。（我们可以利用这个 [在线工具](http://inloop.github.io/svg2android/) 将矢量图转换成 VectorDrawable）。

最后，如果要在项目中使用 VD，则以下几点需要着重注意：

- 1）、**必须通过 app:arcCompat 属性来使用 svg，如果通过 src，则在低版本手机上会出现不兼容的问题**。
- 2）、可能会**不兼容selector**，在 **Activity** 中手动兼容即可，兼容代码如下所示：

    
   ` static {                     
        AppCompatDelegate.setCompatVectorFromResourcesEnabled(true) 
    }`
    
    
- 3）、**不兼容第三方库**。
- 4）、**性能问题：当Vector比较简单时，效率肯定比Bitmap高，复杂则效率会不如Bitmap**。
- 5）、**不便于管理：建议原则为同目录多类型文件，以前缀区别，不同目录相同类型文件，以意义区分**。


与 **VD** 类似，还有一种矢量图标 **iconFont**，即 **字体图标，图标就在字体文件里面，它看着是个图标，其实却是个文字**。它的 **优势** 有如下三个方面：

- 1）、**同 VD 一样，由于 IconFont 是矢量图标,所以可以轻松解决图标适配问题**。
- 2）、**图标以 .ttf 字体文件的形式存在项目中，而 .ttf 文件一般放在 assets 文件夹下,它的体积很小，可以减小 APK 的体积**。
- 3）、**一套图标资源可以在不同平台使用且资源维护方便**。


它的 **缺点** 也很明显，大致有如下三个方面：

- 1）、**需要自定义 svg 图片，并将其转换为 ttf 文件，图标制作成本比较高**。
- 2）、**添加图标时需要重新制作 ttf 文件**。
- 3）、**只能支持单色，不支持渐变色图标**。


如果你想要使用 **iconfont**，可以在阿里的 [iconfont](https://www.iconfont.cn/) 上寻找资源。此外，**使用 [Android-Iconics](https://github.com/mikepenz/Android-Iconics) 可以在你的应用中便于使用任何的 iconfont 或 .svg 图片作为 drawable**。最后，如果我们 **仅仅想提取仅需要的美化文字，以压缩 assets 下的字体文件大小，可以使用 [FontZip](https://github.com/forJrking/FontZip) 字体提取工具**。

如果不是纯色小 **icon** 类型的图片，则建议使用 **WebP**。只要你的 **App** 的 **minSdkVersion** 高于 14（Android 4.0+） 即可。**WebP** 不仅支持透明度，而且压缩率比 **JPEG** 更高，在相同画质下体积更小。但是，**只有 Android 4.2.1+ 才支持显示含透明度的 WebP**，此外，它的 **兼容性不好，并且不便于预览，需使用浏览器打开**。

对于应用之前就存在的图片，我们可以使用 [PNG转换WebP](https://convertio.co/zh/png-webp/) 的转换工具来进行转换。但是，一个一个转换开发效率太低，因此我们可以 **使用[WebpConvert_Gradle_Plugin](https://github.com/meili/WebpConvert_Gradle_Plugin/blob/master/README.zh-cn.md) 这个 gradle 插件去批量进行转换**，它的实现原理是 **在 mergeXXXResource Task 和 processXXXResource Task 之间插入了一个 WebpConvertPlugin task 去将 png、jpg 图片批量替换成了 webp 图片**。

此外，在 **Gradle** 构建 **APK** 的过程中，我们可以判断当前 **App** 的 **minSdkVersion** 以及图片文件的类型来选用是否能使用 **WebP**，代码如下所示：


    boolean isPNGWebpConvertSupported() {
        if (!isWebpConvertEnable()) {
            return false
        }

        // Android 4.0+
        return GradleUtils.getAndroidExtension(project).defaultConfig.minSdkVersion.apiLevel >= 14
        // 4.0
    }

    boolean isTransparencyPNGWebpConvertSupported() {
        if (!isWebpConvertEnable()) {
            return false
        }

        // Lossless, Transparency, Android 4.2.1+
        return GradleUtils.getAndroidExtension(project).defaultConfig.minSdkVersion.apiLevel >= 18
        // 4.3
    }

    def convert() {
        String resPath = "${project.buildDir}/${AndroidProject.FD_INTERMEDIATES}/res/merged/${variant.dirName}"
        def resDir = new File("${resPath}")
        resDir.eachDirMatch(~/drawable[a-z0-9-]*/) { dir ->
            FileTree tree = project.fileTree(dir: dir)
            tree.filter { File file ->
                return (isJPGWebpConvertSupported() && (file.name.endsWith(SdkConstants.DOT_JPG) || file.name.endsWith(SdkConstants.DOT_JPEG))) || (isPNGWebpConvertSupported() && file.name.endsWith(SdkConstants.DOT_PNG) && !file.name.endsWith(SdkConstants.DOT_9PNG))
            }.each { File file ->
                def shouldConvert = true
                if (file.name.endsWith(SdkConstants.DOT_PNG)) {
                    if (!isTransparencyPNGWebpConvertSupported()) {
                        shouldConvert = !Imaging.getImageInfo(file).isTransparent()
                    }
                }
                if (shouldConvert) {
                    WebpUtils.encode(project, webpFactorQuality, file.absolutePath, webp)
                }
            }
        }
    }   


最后，这里再补充下在平时项目开发中对 **图片放置优化的大概思路**，如下所示：

- 1）、**聊天表情出一套图 => hdpi**。
- 2）、**纯色小 icon 使用 VD => raw**。
- 3）、**背景大图出一套 => xhdpi**。
- 4）、**logo 等权重比较大的图片出两套 => hdpi，xhdpi**。
- 5）、**若某些图在真机中有异常，则用多套图**。
- 6）、**若遇到奇葩机型，则针对性补图**。

然后，我们来讲解下资源如何进行混淆。


## 5、资源混淆

同代码混淆类似，资源混淆将 **资源路径混淆成单个资源的路径**，这里我们可以使用 **AndroidResGuard**，它可以使冗余的资源路径变短，例如将 **res/drawable/wechat** 变为 **r/d/a**。

[AndroidResGuard 项目地址](https://github.com/shwenzhang/AndResGuard/blob/master/README.zh-cn.md)

下面，我们就使用 **AndroidResGuard** 来对资源进行混淆。

### 1、AndroidResGuard 实战

#### 1、首先，我们在项目的根 build.gradle 文件下加入下面的插件依赖：


    classpath 'com.tencent.mm:AndResGuard-gradle-plugin:1.2.17'
    
    
#### 2、然后，在项目 module 下的 build.gradle 文件下引入其插件：

    
    apply plugin: 'AndResGuard'


#### 3、接着，加入 AndroidResGuard 的配置项，如下是默认设置好的配置：


    andResGuard {
        // mappingFile = file("./resource_mapping.txt")
        mappingFile = null
        use7zip = true
        useSign = true
        // 打开这个开关，会keep住所有资源的原始路径，只混淆资源的名字
        keepRoot = false
        // 设置这个值，会把arsc name列混淆成相同的名字，减少string常量池的大小
        fixedResName = "arg"
        // 打开这个开关会合并所有哈希值相同的资源，但请不要过度依赖这个功能去除去冗余资源
        mergeDuplicatedRes = true
        whiteList = [
            // for your icon
            "R.drawable.icon",
            // for fabric
            "R.string.com.crashlytics.*",
            // for google-services
            "R.string.google_app_id",
            "R.string.gcm_defaultSenderId",
            "R.string.default_web_client_id",
            "R.string.ga_trackingId",
            "R.string.firebase_database_url",
            "R.string.google_api_key",
            "R.string.google_crash_reporting_api_key"
        ]
        compressFilePattern = [
            "*.png",
            "*.jpg",
            "*.jpeg",
            "*.gif",
        ]
        sevenzip {
            artifact = 'com.tencent.mm:SevenZip:1.2.17'
            //path = "/usr/local/bin/7za"
        }

        /**
        * 可选： 如果不设置则会默认覆盖assemble输出的apk
        **/
        // finalApkBackupPath = "${project.rootDir}/final.apk"

        /**
        * 可选: 指定v1签名时生成jar文件的摘要算法
        * 默认值为“SHA-1”
        **/
        // digestalg = "SHA-256"
    }


#### 4、最后，我们点击右边的项目 module/Tasks/andresguard/resguardRelease 即可生成资源混淆过的 APK。如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c46b29152e5540bebae35d455639b698~tplv-k3u1fbpfcp-zoom-1.image)


**APK** 生成目录如下：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4273cbf645294b3284ff84540291ce6e~tplv-k3u1fbpfcp-zoom-1.image)


对于 AndResGuard 工具，主要有 **两个功能**，一个是 **资源混淆**，一个是 **资源的极限压缩**。下面，我们就来分别了解下它们的实现原理。


### 2、AndResGuard 的资源混淆原理

资源混淆工具主要是通过 **短路径的优化**，以达到 **减少  resources.arsc、metadata 签名文件以及 ZIP 文件大小** 的效果，其效果分别如下所示：

- 1）、**resources.arsc：它记录了资源文件的名称与路径，使用混淆后的短路径  res/s/a，可以减少文件的大小**。
- 2）、**metadata 签名文件：签名文件 MANIFEST.MF 与 CERT.SF 需要记录所有文件的路径以及它们的哈希值，使用短路径可以减少这两个文件的大小**。
- 3）、**ZIP 文件：ZIP 文件格式里面通过其索引记录了每个文件 Entry 的路径、压缩算法、CRC、文件大小等等信息。短路径的优化减少了记录文件路径的字符串大小**。


### 3、AndResGuard 的极限压缩原理

**AndResGuard** 使用了 **7-Zip 的大字典优化**，**APK** 的 **整体压缩率可以提升 3% 左右**，并且，它还支持针对 resources.arsc、PNG、JPG 以及 GIF 等文件进行强制压缩（在编译过程中，这些文件默认不会被压缩）。那么，**为什么 Android 系统不会去压缩这些文件呢**？主要基于以下 **两点原因**：

- 1）、**压缩效果不明显**：上述格式的文件大部分已经被压缩过，因此，重新做 **Zip** 压缩效果并不明显。比如 重新压缩 **PNG** 和 **JPG** 格式只能减少 **3%～5%** 的大小。
- 2）、**基于读取时间和内存的考虑**：针对于 **没有进行压缩的文件，系统可以使用 mmap 的方式直接读取**，而不需要一次性解压并放在内存中。


此外，抖音 Android 团队还开源了针对于海外市场 App Bundle APK 的 [AabResGuard 资源混淆工具](https://github.com/bytedance/AabResGuard/blob/develop/wiki/zh-cn/README.md)，对它的实现原理有兴趣的同学可以去了解下。然后，我们再看看资源瘦身的其它方案。


## 6、R Field 的内联优化

我们可以通过内联 **R Field** 来进一步对代码进行瘦身，此外，它也**解决了 R Field 过多导致 MultiDex 65536 的问题**。要想实现内联 **R Field**，我们需要 **通过 Javassist 或者 ASM 字节码工具在构建流程中内联 R Field**，其代码如下所示：


    ctBehaviors.each { CtBehavior ctBehavior ->
        if (!ctBehavior.isEmpty()) {
            try {
                ctBehavior.instrument(new ExprEditor() {
                    @Override
                    public void edit(FieldAccess f) {
                        try {
                            def fieldClassName =  JavassistUtils.getClassNameFromCtClass(f.getCtClass())
                            if (shouldInlineRField(className, fieldClassName) && f.isReader()) {
                                def temp = fieldClassName.substring(fieldClassName.indexOf(ANDROID_RESOURCE_R_FLAG) + ANDROID_RESOURCE_R_FLAG.length())
                                def fieldName = f.fieldName
                                def key = "${temp}.${fieldName}"

                                if (resourceSymbols.containsKey(key)) {
                                    Object obj = resourceSymbols.get(key)
                                    try {
                                        if (obj instanceof Integer) {
                                            int value = ((Integer) obj).intValue()
                                            f.replace("\$_=${value};")
                                        } else if (obj instanceof Integer[]) {
                                            def obj2 = ((Integer[]) obj)
                                            StringBuilder stringBuilder = new StringBuilder()
                                            for (int index = 0; index < obj2.length; ++index) {
                                                stringBuilder.append(obj2[index].intValue())
                                                if (index != obj2.length - 1) {
                                                    stringBuilder.append(",")
                                                }
                                            }
                                            f.replace("\$_ = new int[]{${stringBuilder.toString()}};")
                                        } else {
                                            throw new GradleException("Unknown ResourceSymbols Type!")
                                        }
                                    } catch (NotFoundException e) {
                                        throw new GradleException(e.message)
                                    } catch (CannotCompileException e) {
                                        throw new GradleException(e.message)
                                    }
                                } else {
                                    throw new GradleException("******** InlineRFieldTask unprocessed ${className}, ${fieldClassName}, ${f.fieldName}, ${key}")
                                }
                            }
                        } catch (NotFoundException e) {
                        }
                    }
                })
            } catch (CannotCompileException e) {
            }
        }
    }
    
    
这里，我们可以 **直接使用蘑菇街的 [ThinRPlugin](https://github.com/meili/ThinRPlugin/blob/master/README.zh-cn.md)**。它的实现原理为：**android 中的 R 文件，除了 styleable 类型外，所有字段都是 int 型变量/常量，且在运行期间都不会改变。所以可以在编译时，记录 R 中所有字段名称及对应值，然后利用 ASM 工具遍历所有 Class，将除 R$styleable.class 以外的所有 R.class 删除掉，并且在引用的地方替换成对应的常量**，从而达到缩减包大小和减少 **Dex** 个数的效果。此外，最近 **ByteX** 也增加了 [shrink_r_class](https://github.com/bytedance/ByteX/blob/master/shrink-r-plugin/README-zh.md) 的 **gradle** 插件，它不仅可以在编译阶段对 **R** 文件常量进行内联，而且还可以 **针对 App 中无用 Resource 和无用 assets 的资源进行检查**。


## 7、资源合并方案

我们可以把所有的资源文件合并成一个大文件，而 **一个大资源文件就相当于换肤方案中的一套皮肤**。它的效果 **比资源混淆的效果会更好**，但是，在此之前，必须要解决 **解析资源** 与 **管理资源** 的问题。其相应的解决方案如下所示：

- **模拟系统实现资源文件的解析：我们需要使用自定义的方式把 PNG、JPG 以及 XML 文件转换为 Bitmap 或者 Drawable**。
- **使用 mmap 加载大资源与资源缓存池管理资源：使用 mmap 加载大资源的方式可以充分减少启动时间与系统内存的占用。而且，需要使用 Glide 等图片框架的资源缓存池 ResourceCache 去释放不再使用的资源文件**。


## 8、资源文件最少化配置

我们需要 **根据 App 目前所支持的语言版本去选用合适的语言资源**，例如使用了 **AppCompat**，如果不做任何配置的话，最终 **APK** 包中会包含 **AppCompat** 中所有已翻译语言字符串，无论应用的其余部分是否翻译为同一语言。对此，我们可以 **通过 resConfig 来配置使用哪些语言，从而让构建工具移除指定语言之外的所有资源**。同理，也可以**使用 resConfigs 去配置你应用需要的图片资源文件类，如 "xhdpi"、"xxhdpi" 等等**，代码如下所示：


    android {
	    ...
        defaultConfig {
    	    ...
            resConfigs "zh", "zh-rCN"
            resConfigs "nodpi", "hdpi", "xhdpi", "xxhdpi", "xxxhdpi"
        }
        ...
    }    
    
    
此外，我们还以 **利用 Density Splits 来选择应用应兼容的屏幕尺寸大小**，代码如下所示：


    android {
        ...
        splits {
            density {
                enable true
                exclude "ldpi", "tvdpi", "xxxhdpi"
                compatibleScreens 'small', 'normal', 'large', 'xlarge'
            }
        }
        ...
    }
    
    
## 9、尽量每张图片只保留一份

比如说，我们统一只把图片放到 **xhdpi** 这个目录下，那么 **在不同的分辨率下它会做自动的适配**，即 **等比例地拉伸或者是缩小**。


## 10、资源在线化

我们可以 **将一些图片资源放在服务器**，然后 **结合图片预加载** 的技术手段，这些 **既可以满足产品的需要，同时可以减小包大小**。


## 11、统一应用风格

如设定统一的 **字体、尺寸、颜色和按钮按压效果、分割线 shape、selector 背景** 等等。


# 四、So 瘦身方案探索

对于主要由 **C/C++** 实现的 **Native Library** 而言，常规的优化方式就是 **去除 Debug 信息，使用 C++_shared 等等**。下面，对于 **So** 瘦身，我们看看还有哪些方案。


## 1、So 移除方案

**So** 是 **Android** 上的动态链接库，在我们 **Android** 应用开发过程中，有时候 **Java** 代码不能满足需求，比如一些 **加解密算法或者音视频编解码功能**，这个时候就必须要通过 **C** 或者是 **C++** 来实现，之后生成 **So** 文件提供给 **Java** 层来调用，在生成 **So** 文件的时候就需要考虑生成市面上不同手机 **CPU** 架构的文件。目前，**Android** 一共 **支持7种不同类型的 CPU 架构**，比如常见的 **armeabi、armeabi-v7a、X86** 等等。理论上来说，**对应架构的 CPU 它的执行效率是最高的**，但是这样会导致 **在 lib 目录下会多存放了各个平台架构的 So 文件**，所以 **App** 的体积自然也就更大了。

因此，我们就需要对 **lib** 目录进行缩减，我们 **在 build.gradle 中配置这个 abiFiliters 去设置 App 支持的 So 架构**，其配置代码如下所示：

    
    defaultConfig {
        ndk {
            abiFilters "armeabi"
        }
    }


**一般情况下，应用都不需要用到 neon 指令集，我们只需留下 armeabi 目录就可以了**。因为 **armeabi 目录下的 So 可以兼容别的平台上的 So**，相当于是一个万金油，都可以使用。但是，这样 **别的平台使用时性能上就会有所损耗，失去了对特定平台的优化**。


## 2、So 移除方案优化版

上面我们说到了想要完美支持所有类型的设备代价太大，那么，我们能不能采取一个 **折中的方案**，就是 **对于性能敏感的模块，它使用到的 So，我们都放在 armeabi 目录当中随着 Apk 发出去，然后我们在代码中来判断一下当前设备所属的 CPU 类型，根据不同设备 CPU 类型来加载对应架构的 So 文件**。这里我们举一个小栗子，比如说我们 **armeabi** 目录下也加上了 **armeabi-v7** 对应的 **So**，然后我们就可以在代码当中做判断，如果你是 **armeabi-v7** 架构的手机，那我们就直接加载这个 **So**，以此达到最佳的性能，这样包体积其实也没有增加多少，同时也实现了高性能的目的，比如 **微信和腾讯视频 App** 里面就使用了这种方式，如下图所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99984edb5b4a400fbf5b49d57cb14010~tplv-k3u1fbpfcp-zoom-1.image)


看到上图中的 **libimagepipeline_x86.so**，下面我们就以这个 so 为例来写写加载它的伪代码，如下所示：


    String abi = "";
    // 获取当前手机的CPU架构类型
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
        abi = Buildl.CPU_ABI;
    } else {
        abi = Build.SUPPORTED_ABIS[0];
    }
    
    if (TextUtils.equals(abi, "x86")) {
        // 加载特定平台的So
        
    } else {
        // 正常加载
        
    }


接下来，我们再了解下 **So** 优化当中别的优化方式。


## 3、使用 XZ Utils 对 Native Library 进行压缩

**Native Library** 同 **Dex** 一样,也可以使用 **XZ Utils** 进行压缩，对于 **Native Library** 的压缩，我们 **只需要去加载启动过程相关的 Library，而其它的都可以在应用首次启动时进行解压**，并且，**压缩效果与 Dex 压缩的效果是相似的**。

此外，关于 **Nativie Library 压缩之后的解压，我们也可以使用 Facebook 的 so 加载库 [SoLoader](https://github.com/facebook/SoLoader)**，它 **能够解压应用的 Native Library 并能递归地加载在 Android 平台上不支持的依赖项**。由于这套方案对启动时间的影响比较大，所以先把它压箱底下吧。


## 4、对 Native Library 进行合并

**在 Android 4.3（API 17) 之前，单个进程加载的 SO 数量是有限制的，在 Google 的 linker.cpp 源码中有很明显的定义**，如下图所示：


![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/545639a5ec32454db4f6e68bf3bc398a~tplv-k3u1fbpfcp-zoom-1.image)


为了解决这个问题，**FaceBook** 写了一个 [合并Native Library的demo](https://github.com/fbsamples/android-native-library-merging-demo)，我们可以 **按照自身 App 的 so 情况来配置需要合并哪些对象。由于合并共享对象（即 .so 文件）在原先的构建流程中是无法实现的，因此 FaceBook 更改了链接库的方式，并把它集成到了构建系统 [Buck](https://github.com/facebook/buck) 中。该功能允许每个应用程序指定应合并的 .so 库，从而避免意外引入不必要的依赖关系。然后，Buck 负责为每个合并的 .so 库收集所有对象（文件），并将它们与适当的依赖项链接在一起。**


## 5、删除 Native Library 中无用的导出 symbol

我们可以去 **分析代码中的 JNI 方法以及不同 Library 库的方法调用，然后找出无用的 symbol 并删除，这样 Linker 在编译的时候也会把 symbol 对应的无用代码给删除**。在 **Buck 有 [NativeRelinker](https://github.com/facebook/buck/blob/master/src/com/facebook/buck/android/relinker/NativeRelinker.java)** 这个类，它就实现了这个功能，其 **类似于 Native Library 的 ProGuard Shrinking 功能**。

至此，可以看到，**FaceBook 出品的 Buck 同 ReDex 一样，里面的功能都十分强大，Buck 除了实现 Library Merge 和 Relinker 功能之外，还实现了三大功能**，如下所示：

- 1）、多语言拆分
- 2）、分包支持
- 3）、ReDex 支持


如果有相应需求或对 Buck 感兴趣的同学可以去看看它们的实现源码。


## 6、So 动态下载

我们可以 **将部分 So 文件使用动态下发的形式进行加载**。也就是在业务代码操作之前，我们可以先从服务器下载下来 So，接下来再使用，这样包体积肯定会减少不小。但是，如果要把这项技术 **稳定落地到实际生产项目中需要解决一些问题**，具体的 so  动态化关键技术点和需要避免的坑可以参见 [动态下发 so 库在 Android APK 安装包瘦身方面的应用
](https://mp.weixin.qq.com/s/X58fK02imnNkvUMFt23OAg)，这里就不多赘述了。


# 五、其它优化方案

## 1、插件化

我们可以使用插件化的手段 **对代码结构进行调整**，如果我们 **App** 当中的每一个功能都是一个插件，并且都是可以从服务器下发下来的，那 **App** 的包体积肯定会小很多。插件化相关的知识非常多而且不属于我们的重点，并且，插件化严格来说属于 **基础架构研发** 这块的知识，掌握它是**成为 Android 架构师的必经之路**，关于 **Android 架构师的学习路线** 可以参考 [Awesome-Android-Architecture](https://github.com/JsonChao/Awesome-Android-Architecture)，预计今年会完成部分学习内容，敬请期待。


## 2、业务梳理

我们需要 **回顾过去的业务**，合理地去 **评估并删除无用或者低价值的业务**。


## 3、转变开发模式

如果所有的功能都不能移除，那就可能需要去转变开发模式，比如可以更多地 **采用 H5、小程序** 这样开发模式。


# 六、包体积监控

对于应用包体积的监控，也应该和内存监控一样，去作为正式版本的发布流程中的一环，并且应该 **尽量地去实现自动化与平台化**。（这里建议 **任何大于 100kb 的功能都需要审批**，特别是需要引入第三方库时，更应该慎重）

## 1、包体积监控的纬度

包体积的监控，主要可以从如下 **三个纬度** 来进行：


- 1）、**大小监控：通常是记录当前版本与上一个或几个版本直接的变化情况，如果当前版本体积增长较大，则需要分析具体原因，看是否有优化空间**。
- 2）、**依赖监控：包括J ar、aar 依赖**。
- 3）、**规则监控：我们可以把包体积的监控抽象为无用资源、大文件、重复文件、R  文件等这些规则**。


包体积的 **大小监控** 和 **依赖监控** 都很容易实现，而要实现 **规则监控** 却得花不少功夫，幸运的是 **Matrix** 中的   [ApkChecker](https://github.com/Tencent/matrix/wiki/Matrix-Android-ApkChecker)  就实现了包体积的规则监控，其 **使用文档与实现原理** 微信团队已经写得很清楚了，这里就不再一一赘述，有兴趣的同学可以去研究下。


# 七、瘦身优化常见问题

瘦身优化是性能优化当中不那么重要的一个分支，不过对于处于稳定运营期的产品会比较有帮助。下面我们就来看看对于瘦身优化有哪些常见问题。


## 1、怎么降低 Apk 包大小？

我们在回答的时候要注意一些 **可操作的干货**，同时注意结合你的 **项目周期**。主要可以从以下 **三点** 来回答：
    

- 1）、**代码：Proguard、统一三方库、无用代码删除**。
- 2）、**资源：无用资源删除、资源混淆**。
- 3）、**So：只保留 Armeabi、更优方案**。
    

在项目初期，我们一直在不断地加功能，加入了很多的代码、资源，同时呢，也没有相应的规范，所以说，**UI** 同学给我们很多 **UI** 图的时候，都是没有经过压缩的图片，长期累积就会导致我们的包体积越来越大。到了项目稳定期的时候，我们对各种运营数据进行考核，发现 **APK** 的包大小影响了用户下载的意愿，于是我们就着手做包体积的优化，我们采用的是 **Android Studio 自带的 Analyze APK 来做的包体积分析**，主要就是做了代码、资源、So 等三个方面的重点优化。

首先，针对于代码瘦身，第一点，我们首先 **使用 Proguard 工具进行了混淆**，它将程序代码转换为功能相同，但是不容易理解的形式。比如说将一个很长的类转换为字母 a，同时，这样做还有一个好处，就是让代码更加安全了。第二点呢，我们将项目中使用到的一些 **第三方库进行了统一**，比如说图片库、网络库、数据库等，不允许项目中出现功能相同，但是却实现不一样的库。同时也做了 **规范，之后引入的三方库，需要去考量它的大小、方法数等，而且呢，如果只是需要一个很大库的一个小功能，那我们就修改源码，只引入部分代码即可**。第三点，我们将项目中的 **无用代码进行了删减**，我们**使用了 AOP 的方式统计到了哪些 Activity 以及 fragment 在真实的场景下没有用户使用**，这样你就可以删除掉了。**对于那些不是 Activity 或者是 Fragment 的类，我们切了很多类的构造函数**，这样你就可以统计出来这些类在线上有没有真正被调用到。但是，**对于代码的瘦身效果，实际上不是很明显**。

接下来，我们做了资源的瘦身。首先，我们 **移除了项目当中冗余的资源文件**，这一点在项目当中一定会遇到。然后，我们做了 **资源图片的压缩，UI 同学给我们资源图片的时候，需要确认已经是压缩过的图片**，同时，我们还会做一个 **兜底策略，在打包的时候，如果图片没有被压缩过，那我们就会再来压缩一遍**，这个效果就非常的明显。对于资源，我们还做了 **资源的混淆**，也就是将冗余的资源名称换成简短的名字，**资源压缩的效果要比代码瘦身的效果要好的多**。

最后，我们做了 **So** 的瘦身。首先，我们**只保留了 armeabi 这个目录**，它可以 **兼容别的 CPU 架构，这点的优化效果非常的明显**。移除了对别的架构适配 So 之后，我们还做了另外一个处理，**对于项目当中使用到的视频模块的 So，它对性能要求非常高，所以我们采用了另外一种方式，我们将所有这个模块下的 So 都放到了 armeabi 这个目录下，然后在代码中做判断，如果是别的 CPU 架构，那我们就加载对应 CPU 架构的 So 文件即可**。这样即减少了包体积，同时又达到了性能最佳。最后，通过实践可以看出 S**o瘦身的效果一般是最好的**。


## 2、Apk 瘦身如何实现长效治理？

主要可以从以下 **两个方面** 来进行回答：

- 1）、**发版之前与上个版本包体积对比，超过阈值则必须优化**。
- 2）、**推进插件化架构改进**。


在大型项目中，最好的方式就是 **结合 CI**，每个开发同学 **在往主干合入代码的时候需要经过一次预编译，这个预编译出来的包对比主干打出来的包大小，如果超过阈值则不允许合入，需要提交代码的同学自己去优化去提交的代码**。此外，针对项目的 **架构**，我们可以做 **插件化的改造，将每一个功能模块都改造成插件，以插件的形式来支持动态下发，这样应用的包体积就可以从根本上变小了**。


# 八、总结

在本篇文章中，我们主要从以下 **七个方面** 讲解了 **Android** 包体积优化相关的知识：

- 1）、**瘦身优化及 Apk 分析方案：瘦身优势、APK 组成、APK 分析**。
- 2）、**代码瘦身方案探索：Dex 探秘、ProGuard、D8 与 R8 优化、去除 debug 信息与行号信息、Dex 分包优化、使用 XZ Utils 进行 Dex 压缩、三方库处理、移除无用代码、避免产生 Java access 方法、利用 ByteX Gradle  插件平台中的代码优化插件**。
- 3）、**资源瘦身方案探索：冗余资源优化、重复资源优化、图片压缩、使用针对性的图片格式、资源混淆、R Field 的内联优化、资源合并方案、资源文件最少化配置、尽量每张图片只保留一份、资源在线化、统一应用风格**。
- 4）、**So 瘦身方案探索：So 移除方案、So 移除方案优化版、使用 XZ Utils 对 Native Library 进行压缩、对 Native Library 进行合并、删除 Native Library 中无用的导出 symbol、So 动态下载**。
- 5）、**其它优化方案：插件化、业务梳理、转变开发模式**。
- 6）、**包体积监控**。
- 7）、**瘦身优化常见问题**。


如果要想对包体积做更深入的优化，我们就必须对 **APK 组成，Dex、So 动态库以及 Resource 文件格式，还有 APK 的编译流程** 有深入地了解，这样我们才能有 **足够的内功素养** 去实现包体积的深度优化。此外，在做性能优化过程中，为了提升研发效率，降低研发成本，我渐渐发现 **AOP 编译插桩、Gradle 自动化构建** 的知识越来越重要；并且，一旦涉及 **Native** 层甚至 **Android** 内核层的深度优化时，就越发感觉到功力不足。因此，为了 **补充深度优化所需的养分**，从下篇开始，笔者将暂停更新 **深入探索 Android 性能优化系列文章**。下篇文章，笔者的分享将会从先从 **编译插桩** 相关的知识开始，敬请期待~


### 参考链接：
---
1、[Top团队大牛带你玩转Android性能分析与优化 第10章 App瘦身优化](https://coding.imooc.com/learn/list/308.html)

2、[极客时间之Android开发高手课 包体积优化](https://time.geekbang.org/column/article/81202)

3、《Android性能优化最佳实践》第七章 安装包大小优化

4、[android-arscblamer](https://github.com/google/android-arscblamer)

5、[Android SVG to VectorDrawable](http://inloop.github.io/svg2android/)

6、[App瘦身最佳实践](https://www.jianshu.com/p/8f14679809b3)

7、[使用Simian工具扫描重复代码](https://blog.csdn.net/Love667767/article/details/53558382)

8、[FontZip](https://github.com/forJrking/FontZip)

9、[Android-Iconics](https://github.com/mikepenz/Android-Iconics)

10、[iconfont](https://www.iconfont.cn/)

11、[IconFont在Android中的使用](https://www.jianshu.com/p/ba1d076a1e31)

12、[TinyPngPlugin](https://github.com/Deemonser/TinyPngPlugin)

13、[动态下发 so 库在 Android APK 安装包瘦身方面的应用
](https://mp.weixin.qq.com/s/X58fK02imnNkvUMFt23OAg)

14、[Apktool Install Instructions](https://ibotpeaches.github.io/Apktool/install/)

15、[nimbledroid](https://nimbledroid.com/)

16、[android-classyshark](https://github.com/google/android-classyshark/releases)

17、[Android混淆从入门到精通](https://www.jianshu.com/p/7436a1a32891)

18、[APK Expansion Files](https://developer.android.com/google/play/expansion-files)

19、[写入放大](https://zh.wikipedia.org/wiki/%E5%86%99%E5%85%A5%E6%94%BE%E5%A4%A7)

20、[D8 dexer and R8 shrinker](https://r8.googlesource.com/r8)

21、[Android新Dex编译器D8与新混淆工具R8](https://blog.dreamtobe.cn/android_d8_r8/)

22、[Comparison of ProGuard vs. R8: October 2019 edition](https://www.guardsquare.com/en/blog/comparison-proguard-vs-r8-october-2019-edition)

23、[支付宝 App 构建优化解析：Android 包大小极致压缩](https://mp.weixin.qq.com/s/_gnT2kjqpfMFs0kqAg4Qig)

24、[Redex](https://github.com/facebook/redex)

25、[Redex 初探与 Interdex：Andorid 冷启动优化](https://mp.weixin.qq.com/s?__biz=MzI0MjE3OTYwMg==&mid=2649548591&idx=2&sn=cb9118ac7d4ef19c219b3ecb017f650e&chksm=f1180c52c66f8544bb06c7d0715cfadddff028078669f6377dc0df4ccee1a4b366101674de4a&scene=27#wechat_redirect)

26、[Dalvik 可执行文件格式](https://source.android.com/devices/tech/dalvik/dex-format?hl=zh_cn#string-item)

27、[InterDex.cpp 贪心算法部分](https://github.com/facebook/redex/blob/master/opt/interdex/InterDex.cpp#L619)

28、[CrossDexRefMinimizer.cpp 跨dex调用优化](https://github.com/facebook/redex/blob/master/opt/interdex/CrossDexRefMinimizer.cpp)

29、[我是如何通过 nimbledroid 做android app性能优化的](https://www.jianshu.com/p/0d2a582aa7d3)

30、[XZ Embedded](https://tukaani.org/xz/embedded.html)

31、[oatmeal](https://github.com/facebook/SoLoader)

32、[SoLoader](https://github.com/google/android-classyshark/releases)

33、[buck](https://github.com/facebook/buck)

34、[android-native-library-merging-demo](https://github.com/fbsamples/android-native-library-merging-demo)

35、[Redex优化demo](https://github.com/AndroidAdvanceWithGeektime/Chapter22)

36、[android-chunk-utils](https://github.com/madisp/android-chunk-utils)

37、[ResourceUsageAnalyzer.java](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/gradle-core/src/main/java/com/android/build/gradle/tasks/ResourceUsageAnalyzer.java)

38、[Android安装包相关知识汇总](https://tech.meituan.com/2017/04/07/android-shrink-overall-solution.html)

39、[安装包立减1M--微信Android资源混淆打包工具
](https://mp.weixin.qq.com/s/6YUJlGmhf1-Q-5KMvZ_8_Q)

40、[AndResGuard](https://github.com/shwenzhang/AndResGuard)

41、[Android APK 签名原理](https://cloud.tencent.com/developer/article/1354380)

42、[西瓜视频apk瘦身之 Java access 方法删除](https://mp.weixin.qq.com/s/ZHisCVjO_ZrtvvEWBYUQFQ)


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e452e0996ed543a38ef1747294f11065~tplv-k3u1fbpfcp-zoom-1.image)


# Contanct Me

##  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

##  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/643bc5073ac0487f830f09b4f7347b8c~tplv-k3u1fbpfcp-zoom-1.image)
        

##  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


## About me

- ### Email: [chao.qu521@gmail.com]()
- ### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- ### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。