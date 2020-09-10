---

		title:  深入探索Android布局优化（上）
		date: 2020/1/13 22:02:00   
		tags: 
		- 性能优化
		categories: 性能优化
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---


# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

Android的绘制优化其实可以分为两个部分，即**布局(UI)优化和卡顿优化**，而布局优化的核心问题就是要解决因布局渲染性能不佳而导致应用卡顿的问题，所以它可以认为是卡顿优化的一个子集。对于Android开发来说，写布局可以说是一个比较简单的工作，但是如果想将写的每一个布局的渲染性能提升到比较好的程度，要付出的努力是要远远超过写布局所付出的。由于布局优化这一主题包含的内容太多，因此，笔者将它分为了上、中、下三篇，本篇，即为深入探索Android布局优化的上篇。本篇包含的主要内容如下所示：

- 1、绘制原理
- 2、屏幕适配
- 3、优化工具


说到Android的布局绘制，那么我们就不得不先从布局的绘制原理开始说起。

# 一、绘制原理

**Android的绘制实现主要是借助CPU与GPU结合刷新机制共同完成的。**

## 1、CPU与GPU

- CPU负责计算显示内容，包括Measure、Layout、Record、Execute等操作。在UI绘制上的缺陷在于容易显示重复的视图组件，这样不仅带来重复的计算操作，而且会占用额外的GPU资源。
- GPU负责栅格化（**用于将UI元素绘制到屏幕上，即将UI组件拆分到不同的像素上显示**）。

这里举两个栗子来讲解一些CPU和GPU的作用：

- 1、文字的显示首先经过CPU换算成纹理，然后再传给GPU进行渲染。
- 2、而图片的显示首先是经过CPU的计算，然后加载到内存当中，最后再传给GPU进行渲染。

那么，软件绘制和硬件绘制有什么区别呢？我们先看看下图：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05b42e04330a42a8be01de5ff6e4cb4b~tplv-k3u1fbpfcp-zoom-1.image)


这里软件绘制使用的是Skia库（一款在低端设备如手机上呈现高质量的 2D 图形的 跨平台图形框架)进行绘制的，而硬件绘制本质上是使用的OpenGl ES接口去利用GPU进行绘制的。OpenGL是一种跨平台的图形API，它为2D/3D图形处理硬件指定了标准的软件接口。而OpenGL ES是用于嵌入式设备的，它是OpenGL规范的一种形式，也可称为其子集。

并且，由于OpenGl ES系统版本的限制，有很多 绘制API 都有相应的 Android API level 的限制，此外，在Android 7.0 把 OpenGL ES 升级到最新的 3.2 版本的时候，还添加了对Vulkan（一套适用于高性能 3D 图形的低开销、跨平台 API）的支持。Vulan作为下一代图形API以及OpenGL的继承者，它的优势在于大幅优化了CPU上图形驱动相关的性能。


## 2、Android 图形系统的整体架构

Android官方的架构图如下：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f48ffba9ad946f5863173c6d223dd90~tplv-k3u1fbpfcp-zoom-1.image)

为了比较好的描述它们之间的作用，我们可以把应用程序图形渲染过程当作一次绘画过程，那么绘画过程中 Android 的各个图形组件的作用分别如下：

- 画笔：Skia 或者 OpenGL。我们可以用 Skia去绘制 2D 图形，也可以用 OpenGL 去绘制 2D/3D 图形。
- 画纸：Surface。**所有的元素都在 Surface 这张画纸上进行绘制和渲染**。在 Android 中，Window 是 View 的容器，每个窗口都会关联一个 Surface。而 WindowManager 则负责管理这些窗口，并且把它们的数据传递给 SurfaceFlinger。
- 画板：Graphic Buffer。**Graphic Buffer 缓冲用于应用程序图形的绘制，在 Android 4.1 之前使用的是双缓冲机制，而在 Android 4.1 之后使用的是三缓冲机制。**
- 显示：SurfaceFlinger。它将 WindowManager 提供的所有 Surface，通过硬件合成器 Hardware Composer 合成并输出到显示屏。

在了解完Android图形系统的整体架构之后，我们还需要了解下Android系统的显示原理，关于这块内容可以参考我之前写的[Android性能优化之绘制优化](https://jsonchao.github.io/2019/07/28/Android%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E4%B9%8B%E7%BB%98%E5%88%B6%E4%BC%98%E5%8C%96/)的Android系统显示原理一节。


## 3、RenderThread

在Android系统的显示过程中，虽然我们利用了GPU的图形高性能计算的能力，但是从计算Display到通过GPU绘制到Frame Buffer都在UI线程中完成，此时如果能让GPU在不同的线程中进行绘制渲染图形，那么绘制将会更加地流畅。

于是，在Android 5.0之后,引入了RenderNode和RenderThread的概念，它们的作用如下：

- RenderNode：进一步封装了Display和某些View的属性。
- RenderThread：渲染线程，负责执行所有的OpenGl命令，其中的RenderNode保存有渲染帧的所有信息，能在主线程有耗时操作的前提下保证动画流畅。

CPU将数据同步给GPU之后，通常不会阻塞等待RenderThread去利用GPU去渲染完视图，而是通知结束之后就返回。加入ReaderThread之后的整个显示调用流程图如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd6cc0115c7447e2b78323b5e5a488f8~tplv-k3u1fbpfcp-zoom-1.image)

在Android 6.0之后，其在adb shell dumpsys gxinfo命令中添加了更加详细的信息，在优化工具一节中我将详细分析下它的使用。

在Android 7.0之后，对HWUI进行了重构，它是用于2D硬件绘图并负责硬件加速的主要模块，其使用了OpenGl ES来进行GPU硬件绘图。此外，Android 7.0还支持了Vulkan，并且，Vulkan 1.1在Android 被引入。

### 硬件加速存在哪些问题？

我们都知道，硬件加速的原理就是将CPU不擅长的图形计算转换成GPU专用指令。

- 1、其中的OpenGl API调用和Graphic Buffer缓冲区至少会占用几MB以上的内存，**内存消耗较大**。
- 2、有些OpenGl的绘制API还没有支持，特别是比较低的Android系统版本，并且由于Android每一个版本都会对渲染模块进行一些重构，导致了在硬件加速绘制过程中会出现一些不可预知的Bug。如在**Android 5.0~7.0机型上出现的libhwui.so崩溃问题**，需要使用inline Hook、GOT Hook等native调试手段去进行分析定位，可能的原因是**ReaderThread与UI线程的sync同步过程出现了差错，而这种情况一般都是有多个相同的视图绘制而导致的，比如View的复用、多个动画同时播放**。


## 4、刷新机制

16ms发出VSync信号触发UI渲染，大多数的Android设备屏幕刷新频率为60HZ，如果16ms内不能完成渲染过程，则会产生掉帧现象。


# 二、屏幕适配

我们都知道，Android手机屏幕的差异化导致了严重的碎片化问题，并且屏幕材质也是用户比较关注的一个重要因素。

首先，我们来了解下主流Android屏幕材质，目前主要有两类：

- LCD（Liquid Crystal Display）：液晶显示器。
- OLED（Organic Light-Emitting Diode ）:有机发光二极管。

早在20世纪60年代，随着半导体集成电路的发展，美国人成功研发出了第一块液晶显示屏LCD，而现在大部分最新的高端机使用的都是OLED材质，这是因为相比于LCD屏幕，OLED屏幕在色彩、可弯曲程度、厚度和耗电等方面都有一定的优势。正因为如此，现在主流的全面屏、曲面屏与未来的柔性折叠屏，使用的几乎都是 OLED 材质。当前，好的材质，它的成本也必然会比较昂贵。

## 1、OLED 屏幕和 LCD 屏幕的区别

如果要明白OLED 屏幕和LCD屏幕的区别，需要了解它们的运行原理，下面，我将分别进行讲解。

### 屏幕的成像原理

**屏幕由无数个点组成，并且，每个点由红绿蓝三个子像素组成，每个像素点通过调节红绿蓝子像素的颜色配比来显示不同的颜色，最终所有的像素点就会形成具体的画面。**

### LCD背光源与OLED自发光

下面，我们来看下LCD和OLED的总体结构图，如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/923d21585a1b43e5bcf31a95f7f419ca~tplv-k3u1fbpfcp-zoom-1.image)

LCD的发光原理主要在于**背光层Back-light**，它通常都会由大量的LED背光灯组成以用于显示白光，之后，**为了显示出彩色，在其上面加了一层有颜色的薄膜**，白色的背光穿透了有颜色的薄膜后就可以显示出彩色了。但是，**为了实现调整红绿蓝光的比例，需要在背光层和颜色薄膜之间加入一个控制阀门，即液晶层liquid crystal，它可以通过改变电压的大小来控制开合的程度，开合大则光多，开合小则光少**。

对于OLED来说，它不需要LCD屏幕的背光层和用于控制出光量的液晶层，它就像一个有着无数个小的彩色灯泡组成的屏幕，只需要给它通电就能发光。

### LCD的致命缺陷

它的液晶层不能完全关合，如果LCD显示黑色，会有部分光穿过颜色层，所以LCD的黑色实际上是白色和黑色混合而成的灰色。而OLED不一样，OLED显示黑色的时候可以直接关闭区域的像素点。

此外，由于背光层的存在，所以LCD显示器的背光非常容易从屏幕与边框之间的缝隙泄漏出去，即会产生显示器漏光现象。

### OLED屏幕的优势

- 1、由于没有有背光层和液晶层的存在，所以它的**厚度更薄，其弯曲程度可以达到180%**。
- 2、对比度（白色比黑色的比值）更高，使其画面颜色越浓；相较于LCD来说，**OLED是油画，色彩纯而细腻，而LCD是水彩笔画，色彩朦胧且淡**。
- 3、OLED每个像素点都是独立的，所以OLED可以单独点亮某些像素点，即能实现**单独点亮**。而LCD只能控制整个背光层的开关。并且，由于OLED单独点亮的功能，使其**耗电程度大大降低**。
- 4、OLED的**屏幕响应时间很快**，不会造成画面残留以致造成视觉上的拖影现象。而LCD则会有严重的拖影现象。


### OLED屏幕的劣势

- 1、由于OLED是**有机材料**，导致其寿命是不如LCD的
有机材料的。并且，由于OLED单独点亮的功能，会使每个像素点工作的时间不一样，这样，在屏幕老化时就会导致色彩显示不均匀，即产生**烧屏**现象。
- 2、由于OLED就不能采取控制电压的方式去调整亮度，所以目前只能通过不断的开关开关开关去进行调光。
- 3、OLED的屏幕像素点排列方式不如LCD的紧凑，所以在分辨率相同的情况下，OLED的屏幕是不如LCD清楚的。即OLED的**像素密度较低**。


## 2、屏幕适配方案

我们都知道，Android 的 系统碎片化、机型以及屏幕尺寸碎片化、屏幕分辨率碎片化非常地严重。所以，一个好的屏幕适配方案是很重要的。接下来，我将介绍目前主流的屏幕适配方案。

### 1、最原始的Android适配方案：dp + 自适应布局或weight比例布局

首先，我们来回顾一下px、dp、dpi、ppi、density等概念：

- px：像素点，px = density * dp。
- ppi：像素密度，每英寸所包含的像素数目，屏幕物理参数，不可调整，dpi没有人为调整时 = ppi。
- dpi：像素密度，在系统软件上指定的单位尺寸的像素数量，可人为调整，dpi没有人为调整时 = ppi。
- dp：density-independent pixels，即密度无关像素，基于屏幕物理分辨率的一个抽象的单位，以dp为尺寸单位的控件，在不同分辨率和尺寸的手机上代表了不同的真实像素，比如在分辨率较低的手机中，可能1dp = 1px,而在分辨率较高的手机中，可能1dp=2px，这样的话，一个64*64dp的控件，在不同的手机中就能表现出差不多的大小了，px = dp * （dpi / 160）。
- denstiy：密度，屏幕上每平方英寸所包含的像素点个数，density = dpi / 160。

通常情况下，我们只需要使用dp + 自适应布局（如鸿神的AutoLayout、ConstraintLayout等等）或weight比例布局即可基本解决碎片化问题，当然，这种方式也存在一些问题，比如**dpi和ppi的差异所导致在同一分辨率手机上控件大小的不同**。


### 2、宽高限定符适配方案

它就是穷举市面上所有的Android手机的宽高像素值，通过设立一个基准的分辨率，其他分辨率都根据这个基准分辨率来计算，在不同的尺寸文件夹内部，根据该尺寸编写对应的dimens文件，如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/034d57f109a546e4bf53103eb7e577a3~tplv-k3u1fbpfcp-zoom-1.image)

比如以480x320为基准分辨率：

- 宽度为320，将任何分辨率的宽度整分为320份，取值为x1-x320。
- 高度为480，将任何分辨率的高度整分为480份，取值为y1-y480。

那么对于800*480的分辨率的dimens文件来说：

- x1=(480/320)*1=1.5px
- x2=(480/320)*2=3px

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5dd9de5e9e314f08b37710038b40d0e2~tplv-k3u1fbpfcp-zoom-1.image)

此时，如果UI设计界面使用的就是基准分辨率，那么我们就可以按照设计稿上的尺寸填写相对应的dimens去引用，而当APP运行在不同分辨率的手机中时，系统会根据这些dimens去引用该分辨率对应的文件夹下面去寻找对应的值。但是这个方案由一个缺点，就是无法做到向下兼容去使用更小的dimens，比如说800x480的手机就一定要找到800x480的限定符，否则就只能用统一默认的dimens文件了。


### 3、UI适配框架AndroidAutoLayout的适配方案

因宽高限定符方案的启发，鸿神出品了一款能使用UI适配更加开发高效和适配精准的项目。

[项目地址](https://github.com/hongyangAndroid/AndroidAutoLayout)

基本使用步骤如下：

第一步：在你的项目的AndroidManifest中注明你的设计稿的尺寸：

    <meta-data android:name="design_width" android:value="768">
    </meta-data>
    <meta-data android:name="design_height" android:value="1280">
    </meta-data>


第二步：让你的Activity继承自AutoLayoutActivity。如果你不希望继承AutoLayoutActivity，可以在编写布局文件时，直接使用AutoLinearLayout、Auto***等适配布局即可。

接下来，直接在布局文件里面使用具体的像素值就可以了，因为在APP运行时，AndroidAutoLayout会帮助我们根据不同手机的具体尺寸按比例伸缩。

AndroidAutoLayout在宽高限定符适配的基础上，解决了其dimens不能向下兼容的问题，但是它在运行时会在onMeasure里面对dimens去做变换，所以对于自定义控件或者某些特定的控件需要进行单独适配；并且，整个UI的适配过程都是由框架完成的，以后想替换成别的UI适配方案成本会比较高，而且，不幸的是，项目已经停止维护了。


### 4、smallestWidth适配方案（sw限定符适配）

smallestWidth即最小宽度，系统会根据当前设备屏幕的 最小宽度 来匹配 values-sw<N>dp。

我们都知道，移动设备都是允许屏幕可以旋转的，当屏幕旋转时，屏幕的高宽就会互换，加上 最小 这两个字，是因为这个方案是不区分屏幕方向的，它只会把屏幕的高度和宽度中值最小的一方认为是 最小宽度。

并且它跟宽高限定符适配原理上是一样，都是系统通过特定的规则来选择对应的文件。**它与AndroidAutoLayout一样，同样解决了其dimens不能向下兼容的问题，如果该屏幕的最小宽度是360dp，但是项目中没有values-sw360dp文件夹的话，它就可能找到values-sw320dp这个文件夹**，其尺寸规则命名如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5cd6c06b559a44cf86192faf3ba72662~tplv-k3u1fbpfcp-zoom-1.image)


假如加入我们的设计稿的像素宽度是375，那么其对应的values-sw360dp和values-sw400dp宽度如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1552759b09d48dbabeda6b754c40a62~tplv-k3u1fbpfcp-zoom-1.image)

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a0428b4c0e6451bb666f19e1c04d44e~tplv-k3u1fbpfcp-zoom-1.image)

smallestWidth的适配机制由系统保证，我们只需要针对这套规则生成对应的资源文件即可，即使对应的smallestWidth值没有找到完全对应的资源文件，它也能向下兼容，寻找最接近的资源文件。虽然多个dimens文件可能导致apk变大，但是其增加大小范围也只是在300kb-800kb这个区间，这还是可以接受的。这套方案唯一的变数就是选择需要适配哪些最小宽度限定符的文件，如果您生成的 values-sw<N>dp 与设备实际的 最小宽度 差别不大，那误差也就在能接受的范围内，如果差别很大，那效果就会很差。最后，总结一下这套方案的优缺点：

**优点：**

- 1、稳定且无性能损耗。
- 2、可通过选择需要哪些最小宽度限定符文件去控制适配范围。
- 3、在自动生成values-sw<N>的插件基础下，学习成本较低。

插件地址为[自动生成values-sw<N>的项目代码](https://github.com/ladingwu/dimens_sw)。生成需要的values-sw<N>dp文件夹的步骤如下：

- 1、clone该项目到本地,以Android项目打开。
- 2、DimenTypes文件中写入你希望适配的sw尺寸，默认的这些尺寸能够覆盖几乎所有手机适配需求。
- 3、DimenGenerator文件中填写设计稿的尺寸(DESIGN_WIDTH是设计稿宽度，DESIGN_HEIGHT是设计稿高度)。
- 4、执行lib module中的DimenGenerator.main()方法，当前地址下会生成相应的适配文件,把相应的文件连带文件夹拷贝到正在开发的项目中。

**缺点：**

- 1、侵入性高，后续切换其他屏幕适配方案需修改大量 dimens 引用。
- 2、覆盖更多不同屏幕的机型需要生成更多的资源文件，使APK体积变大。
- 3、不能自动支持横竖屏切换时的适配，如要支持需使用 values-w<N>dp 或 屏幕方向限定符 再生成一套资源文件，又使APK体积变大。

**如果想让屏幕宽度随着屏幕的旋转而做出改变该怎么办呢？**

此时根据 values-w<N>dp (去掉 sw 中的 s) 去生成一套资源文件即可。

**如果想区分屏幕的方向来做适配该怎么办呢？**

去根据 屏幕方向限定符 生成一套资源文件，后缀加上 -land 或 -port 即可，如：values-sw360dp-land (最小宽度 360 dp 横向)，values-sw400dp-port (最小宽度 720 dp 纵向)。

**注意：**

如果UI设计上明显更适合使用wrap_content,match_parent,layout_weight等,我们就要毫不犹豫的使用，毕竟，上述都是仅仅针对不得不使用固定宽高的情况，我相信基础的UI适配知识大部分开发者还是具备的。如果不具备的话，请看下方：

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0955e91798ce4cb49984a0dd6ba951c7~tplv-k3u1fbpfcp-zoom-1.image" width=30%>
</div>


### 5、今日头条适配方案

它的原理是**根据屏幕的宽度或高度动态调整每个设备的 density (每 dp 占当前设备屏幕多少像素)，通过修改density值的方式，强行把所有不同尺寸分辨率的手机的宽度dp值改成一个统一的值，这样就可以解决所有的适配问题**。其对应的重要公式如下：

    当前设备屏幕总宽度（单位为像素）/  设计图总宽度（单位为 dp) = density

**今日头条适配方案默认项目中只能以高或宽中的一个作为基准来进行适配，并不像 AndroidAutoLayout 一样，高以高为基准，宽以宽为基准，来同时进行适配，为什么？**

因为，现在中国大部分市面上的 Android 设备的屏幕高宽比都不一致，特别是现在的全面屏、刘海屏、弹性折叠屏，使这个问题更加严重，不同厂商推出的手机的屏幕高宽比都可能不一致。所以，我们只能以高或宽其中的一个作为基准进行适配，以此避免布局在高宽比不一致的屏幕上出现变形。

它有以下优势：

- 1、使用成本低，操作简单，使用该方案后在页面布局时不需要额外的代码和操作。
- 2、侵入性低，和项目完全解耦，在项目布局时不会依赖哪怕一行该方案的代码，而且使用的还是 Android 官方的 API，意味着当你遇到什么问题无法解决，想切换为其他屏幕适配方案时，基本不需要更改之前的代码，整个切换过程几乎在瞬间完成，试错成本接近于 0。
- 3、可适配三方库的控件和系统的控件(不止是是 Activity 和 Fragment，Dialog、Toast 等所有系统控件都可以适配)，由于修改的 density 在整个项目中是全局的，所以只要一次修改，项目中的所有地方都会受益。
- 4、不会有任何性能的损耗。
- 5、不涉及私有API。

它的缺点如下所示：

- 1、适配范围不可控，只能一刀切的将整个项目进行适配，这种将所有控件都强行使用我们项目自身的设计图尺寸进行适配的方案会有问题：当某个系统控件或三方库控件的设计图尺寸和和我们项目自身的设计图尺寸差距越大时，该系统控件或三方库控件的适配效果就越差。比较好的解决方案就是按 Activity 为单位，取消当前 Activity 的适配效果，改用其他的适配方案。
- 2、对旧项目的UI适配兼容性不够。

**注意：**

千万不要在此方案上使用smallestWidth适配方案中直接填写设计图上标注的 px 值的做法，这样会使项目强耦合于这个方案，后续切换其它方案都不得不将所有的 layout 文件都改一遍。

这里推荐一下JessYanCoding的[AndroidAutoSize](https://github.com/JessYanCoding/AndroidAutoSize)项目，用法如下：

1、首先在项目的build.gradle中添加该库的依赖：

     implementation 'me.jessyan:autosize:1.1.2'


2、接着 AndroidManifest 中填写全局设计图尺寸 (单位 dp)，如果使用副单位，则可以直接填写像素尺寸，不需要再将像素转化为 dp：

    <manifest>
        <application>            
            <meta-data
                android:name="design_width_in_dp"
                android:value="360"/>
            <meta-data
                android:name="design_height_in_dp"
                android:value="640"/>           
        </application>           
    </manifest>


**为什么只需在AndroidManifest.xml 中填写一下 meta-data 标签就可实现自动运行？**

在 App 启动时，系统会在 App 的主进程中自动实例化声明的 ContentProvider，并调用它的 onCreate 方法，**执行时机比 Application#onCreate 还靠前，可以做一些初始化的工作，这个时候我们就可以利用它的 onCreate 方法在其中启动框架**。如果项目使用了多进程，调用Application#onCreate 中调用下 ContentProvider#query 就能够使用 ContentProvider 在当前进程中进行实例化。


### 6、小结

上述介绍的所有方案并没有哪一个是十分完美的，但我们能清晰的认识到不同方案的优缺点，并将它们的优点相结合，这样才能应付更加复杂的开发需求，创造出最卓越的产品。比如**SmallestWidth 限定符适配方案 主打的是稳定性，在运行过程中极少会出现安全隐患，适配范围也可控，不会产生其他未知的影响，而 今日头条适配方案 主打的是降低开发成本、提高开发效率，使用上更灵活，也能满足更多的扩展需求**。所以，具体情况具体分析，到底选择哪一个屏幕适配方案还是需要去根据我们项目自身的需求去选择。


# 三、优化工具

## 1、Systrace

早在[深入探索Android启动速度优化](https://jsonchao.github.io/2019/11/10/%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%B4%A2Android%E5%90%AF%E5%8A%A8%E9%80%9F%E5%BA%A6%E4%BC%98%E5%8C%96/)一文中我们就了解过Systrace的使用、原理及它作为启动速度分析的用法。而它其实主要是用来分析绘制性能方面的问题。下面我就详细介绍下Systrace作为绘制优化工具有哪些必须关注的点。

### 1）、关注Frames

首先，先在左边栏选中我们当前的应用进程，在应用进程一栏下面有一栏Frames，我们可以看到**有绿、黄、红三种不同的小圆圈**，如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42f5ecdd22894e1087518a517c08ff0a~tplv-k3u1fbpfcp-zoom-1.image)

图中每一个小圆圈代表着当前帧的状态，大致的对应关系如下：

- 正常：绿色。
- 丢帧：黄色。
- 严重丢帧：红色。

并且，**选中其中某一帧**，我们还可以在视图最下方的详情框看到**该帧对应的相关的Alerts报警信息**，以帮助我们去排查问题；此外，如果是大于等于Android 5.0的设备（即API Level21），创建帧的工作工作分为UI线程和render线程。而在Android 5.0之前的版本中，创建帧的所有工作都是在UI线程上完成的。接下来，我们看看该帧对应的详情图，如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80bda8d941df453ca95c8c63ee33fb32~tplv-k3u1fbpfcp-zoom-1.image)

对应到此帧，我们发现这里可能有两个绘制问题：Bitmap过大、布局嵌套层级过多导致的measure和layout次数过多，这就需要我们去在项目中找到该帧对应的Bitmap进行相应的优化，针对布局嵌套层级过多的问题去选择更高效的布局方式，这块后面我们会详细介绍。


### 2）、关注Alerts栏

此外，Systrace的显示界面还在在右边侧栏提供了一栏Alert框去显示出它所检测出所有可能有绘制性能问题的地方及对应的数量，如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01aeb27bef1e444ca760ecf0596c37ab~tplv-k3u1fbpfcp-zoom-1.image)

在这里，我们可以**将Alert框看做是一个是待修复的Bug列表**，通常一个区域的改进可以消除应用程序中的所有类中该类型的警报，所以，不要为这里的警报数量所担忧。

## 2、Layout Inspector

Layout Inspector是AndroidStudio自带的工具，它的主要作用就是用来查看视图层级结构的。

具体的操作路径为：

    点击Tools工具栏 ->第三栏的Layout Inspector -> 选中当前的进程


下面为操作之后打开的[Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid)首页图，如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11547c6f838c451883df09fa6977f33c~tplv-k3u1fbpfcp-zoom-1.image)


其中，**最左侧的View Tree就是用来查看视图的层级结构的**，非常方便，这是它最主要的功能，中间的是一个屏幕截图，最右边的是一个属性表格，比如我在截图中选中某一个TextView（Kotlin/入门及知识点一栏），在属性表格的text中就可以显示相关的信息，如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7092dc9eff184199851a277a81033f50~tplv-k3u1fbpfcp-zoom-1.image)


## 3、Choreographer

Choreographer是用来获取FPS的，并且可以用于线上使用，具备实时性，但是仅能在Api 16之后使用，具体的调用代码如下：

    Choreographer.getInstance().postFrameCallback();
    
使用Choreographer获取FPS的完整代码如下所示：

    private long mStartFrameTime = 0;
    private int mFrameCount = 0;
    
    /**
     * 单次计算FPS使用160毫秒
     */
    private static final long MONITOR_INTERVAL = 160L; 
    private static final long MONITOR_INTERVAL_NANOS = MONITOR_INTERVAL * 1000L * 1000L;
    
    /**
     * 设置计算fps的单位时间间隔1000ms,即fps/s
     */
    private static final long MAX_INTERVAL = 1000L; 

    @TargetApi(Build.VERSION_CODES.JELLY_BEAN)
    private void getFPS() {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN) {
            return;
        }
        Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
            @Override
            public void doFrame(long frameTimeNanos) {
                if (mStartFrameTime == 0) {
                    mStartFrameTime = frameTimeNanos;
                }
                long interval = frameTimeNanos - mStartFrameTime;
                if (interval > MONITOR_INTERVAL_NANOS) {
                    double fps = (((double) (mFrameCount * 1000L * 1000L)) / interval) * MAX_INTERVAL;
                    // log输出fps
                    LogUtils.i("当前实时fps值为： " + fps);
                    mFrameCount = 0;
                    mStartFrameTime = 0;
                } else {
                    ++mFrameCount;
                }

                Choreographer.getInstance().postFrameCallback(this);
            }
        });
    }
    

通过以上方式我们就可以实现实时获取应用的界面的FPS了。但是我们需要排除掉页面没有操作的情况，即只在界面存在绘制的时候才做统计。我们可以通过 addOnDrawListener 去监听界面是否存在绘制行为，代码如下所示：

    
    getWindow().getDecorView().getViewTreeObserver().addOnDrawListener
    

当出现丢帧的时候，我们可以获取应用当前的页面信息、View 信息和操作路径上报至 APM后台，以降低二次排查的难度。此外，我们将连续丢帧超过 700 毫秒定义为冻帧，也就是连续丢帧 42 帧以上。这时用户会感受到比较明显的卡顿现象，因此，我们可以统计更有价值的冻帧率。冻帧率就是计算发生冻帧时间在所有时间的占比。通过解决应用中发生冻帧的地方我们就可以大大提升应用的流畅度。


## 4、Tracer for OpenGL ES 与 GAPID（Graphics API Debugger）

Tracer for OpenGL ES 是 Android 4.1 新增加的工具，它可逐帧、逐函数的记录 App 使用 OpenGL ES 的绘制过程，并且，它可以记录每个 OpenGL 函数调用的消耗时间。当使用Systrace还找不到渲染问题时，就可以去尝试使用它。

而GAPID是 Android Studio 3.1 推出的工具，可以认为是Tracer for OpenGL ES的进化版，它不仅实现了跨平台，而且支持Vulkan与回放。由于它们主要是用于OpenGL相关开发的使用，这里我就不多介绍了。


## 5、自动化测量 UI 渲染性能的方式

**在自动化测试中，我们通常希望通过执行性能测试的自动化脚本来进行线下的自动化检测，那么，有哪些命令可以用于测量UI渲染的性能呢？**

我们都知道，**dumpsys是一款输出有关系统服务状态信息的Android工具**，利用它我们可以获取当前设备的UI渲染性能信息，目前常用的有如下两种命令：

### 1）、gfxinfo

gfxinfo的主要作用是**输出各阶段发生的动画与帧相关的信息**，命令格式如下：

    adb shell dumpsys gfxinfo <PackageName>
    

这里我以[Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid)项目为例，输出其对应的gfxinfo信息如下所示：

    quchao@quchaodeMacBook-Pro ~ % adb shell dumpsys gfxinfo json.chao.com.wanandroid
    Applications Graphics Acceleration Info:
    Uptime: 549887348 Realtime: 549887348
    
    ** Graphics info for pid 1722     [json.chao.com.wanandroid] **
    
    Stats since: 549356564232951ns
    Total frames rendered: 5210
    Janky frames: 193 (3.70%)
    50th percentile: 5ms
    90th percentile: 9ms
    95th percentile: 13ms
    99th percentile: 34ms
    Number Missed Vsync: 31
    Number High input latency: 0
    Number Slow UI thread: 153
    Number Slow bitmap uploads: 6
    Number Slow issue draw commands: 51
    HISTOGRAM: 5ms=4254 6ms=131 7ms=144 8ms=87 9ms=80 10ms=83 11ms=108 12ms=57 13ms=29 14ms=17 15ms=17 16ms=14 17ms=20 18ms=15 19ms=15 20ms=17 21ms=9 22ms=14 23ms=8 24ms=9 25ms=4 26ms=5 27ms=4 28ms=4 29ms=1 30ms=2 31ms=4 32ms=3 34ms=6 36ms=5 38ms=7 40ms=8 42ms=0 44ms=3 46ms=3 48ms=5 53ms=2 57ms=0 61ms=3 65ms=0 69ms=1 73ms=1 77ms=0 81ms=0 85ms=0 89ms=1 93ms=1 97ms=0 101ms=0 105ms=0 109ms=0 113ms=1 117ms=0 121ms=0 125ms=0 129ms=0 133ms=0 150ms=2 200ms=0 250ms=2 300ms=1 350ms=1 400ms=0 450ms=1 500ms=0 550ms=1 600ms=0 650ms=0 700ms=0 750ms=0 800ms=0 850ms=0 900ms=0 950ms=0 1000ms=0 1050ms=0 1100ms=0 1150ms=0 1200ms=0 1250ms=0 1300ms=0 1350ms=0 1400ms=0 1450ms=0 1500ms=0 1550ms=0 1600ms=0 1650ms=0 1700ms=0 1750ms=0 1800ms=0 1850ms=0 1900ms=0 1950ms=0 2000ms=0 2050ms=0 2100ms=0 2150ms=0 2200ms=0 2250ms=0 2300ms=0 2350ms=0 2400ms=0 2450ms=0 2500ms=0 2550ms=0 2600ms=0 2650ms=0 2700ms=0 2750ms=0 2800ms=0 2850ms=0 2900ms=0 2950ms=0 3000ms=0 3050ms=0 3100ms=0 3150ms=0 3200ms=0 3250ms=0 3300ms=0 3350ms=0 3400ms=0 3450ms=0 3500ms=0 3550ms=0 3600ms=0 3650ms=0 3700ms=0 3750ms=0 3800ms=0 3850ms=0 3900ms=0 3950ms=0 4000ms=0 4050ms=0 4100ms=0 4150ms=0 4200ms=0 4250ms=0 4300ms=0 4350ms=0 4400ms=0 4450ms=0 4500ms=0 4550ms=0 4600ms=0 4650ms=0 4700ms=0 4750ms=0 4800ms=0 4850ms=0 4900ms=0 4950ms=0
    Caches:
    Current memory usage / total memory usage (bytes):
    TextureCache          5087048 / 59097600
    Layers total          0 (numLayers = 0)
    RenderBufferCache           0 /  4924800
    GradientCache           20480 /  1048576
    PathCache                   0 /  9849600
    TessellationCache           0 /  1048576
    TextDropShadowCache         0 /  4924800
    PatchCache                  0 /   131072
    FontRenderer A8        184219 /  1478656
        A8   texture 0       184219 /  1478656
    FontRenderer RGBA           0 /        0
    FontRenderer total     184219 /  1478656
    Other:
    FboCache                    0 /        0
    Total memory usage:
    6586184 bytes, 6.28 MB


    Pipeline=FrameBuilder
    Profile data in ms:

	    json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.MainActivity/android.view.ViewRootImpl@4a2142e (visibility=8)
	    json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.ArticleDetailActivity/android.view.ViewRootImpl@4bccbcf (visibility=8)
    View hierarchy:

    json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.MainActivity/android.view.ViewRootImpl@4a2142e
    151 views, 154.02 kB of display lists

    json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.ArticleDetailActivity/android.view.ViewRootImpl@4bccbcf
    19 views, 18.70 kB of display lists


    Total ViewRootImpl: 2
    Total Views:        170
    Total DisplayList:  172.73 kB


下面，我将对其中的关键信息进行分析。

**帧的聚合分析数据**

开始的一栏是统计的当前界面所有帧的聚合分析数据，主要作用是**综合查看App的渲染性能以及帧的稳定性。**

- Graphics info for pid 1722     [json.chao.com.wanandroid] -> 说明了当前提供的是Awesome-WanAndroid应用界面的帧信息，对应的进程id为1722。
- Total frames rendered 5210 -> 本次dump的数据搜集了5210帧的信息。
- Janky frames: 193 (3.70%) -> 5210帧中有193帧发生了Jank，即单帧耗时时间超过了16ms，卡顿的概率为3.70%。
- 50th percentile: 5ms -> 所有帧耗时排序后，其中前50%最大的耗时帧的耗时为5ms。
- 90th percentile: 9ms -> 同上，依次类推。
- 95th percentile: 13ms -> 同上，依次类推。
- 99th percentile: 34ms -> 同上，依次类推。
- Number Missed Vsync: 31 -> 垂直同步失败的帧数为31。
- Number High input latency: 0 -> 处理input耗时的帧数为0。
- Number Slow UI thread: 153 -> 因UI线程的工作而导致耗时的帧数为153。
- Number Slow bitmap uploads: 6 -> 因bitmap加载导致耗时的帧数为6。
- Number Slow issue draw commands: 51 -> 因绘制问题导致耗时的帧数为51。
- HISTOGRAM: 5ms=4254 6ms=131 7ms=144 8ms=87... -> 直方图数据列表，说明了耗时0~5ms的帧数为4254，耗时5~6ms的帧数为131，后续的数据依次类推即可。

后续的log数据表明了不同组件的缓存占用信息，帧的建立路径信息以及总览信息等等，参考意义不大。

可以看到，上述的数据只能让我们总体感受到绘制性能的好坏，并不能去定位具体帧的问题，那么，还有更好的方式去获取具体帧的信息吗？

**添加framestats去获取最后120帧的详细信息**

该命令的格式如下：

    adb shell dumpsys gfxinfo <PackageName> framestats


这里还是以Awesome-WanAndroid项目为例，输出项目标签页的帧详细信息：

    quchao@quchaodeMacBook-Pro ~ % adb shell dumpsys gfxinfo json.chao.com.wanandroid framestats
    Applications Graphics Acceleration Info:
    Uptime: 603118462 Realtime: 603118462

    ...
    
    Window: json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.MainActivity
    Stats since: 603011709157414ns
    Total frames rendered: 3295
    Janky frames: 117 (3.55%)
    50th percentile: 5ms
    90th percentile: 9ms
    95th percentile: 14ms
    99th percentile: 32ms
    Number Missed Vsync: 17
    Number High input latency: 3
    Number Slow UI thread: 97
    Number Slow bitmap uploads: 13
    Number Slow issue draw commands: 20
    HISTOGRAM: 5ms=2710 6ms=75 7ms=81 8ms=70...

    ---PROFILEDATA---
    Flags,IntendedVsync,Vsync,OldestInputEvent,NewestInputEvent,HandleInputStart,AnimationStart,PerformTraversalsStart,DrawStart,SyncQueued,SyncStart,IssueDrawCommandsStart,SwapBuffers,FrameCompleted,DequeueBufferDuration,QueueBufferDuration,
    0,603111579233508,603111579233508,9223372036854775807,0,603111580203105,603111580207688,603111580417688,603111580651698,603111580981282,603111581033157,603111581263417,603111583942011,603111584638678,1590000,259000,
    0,603111595904553,603111595904553,9223372036854775807,0,603111596650344,603111596692428,603111596828678,603111597073261,603111597301386,603111597362376,603111597600292,603111600584667,603111601288261,1838000,278000,
    ...,
    ---PROFILEDATA---

    ...


这里我们只需关注其中的PROFILEDATA一栏，因为它表明了最近120帧每个帧的状态信息。

因为其中的数据是以csv格式显示的，我们将PROFILEDATA中的数据全部拷贝过来，然后放入一个txt文件中，接着，把.txt后缀改为.csv，使用WPS表格工具打开，如下图所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8b35739ed2744d8bbb8cd88bf80ff3c~tplv-k3u1fbpfcp-zoom-1.image)

从上图中，我们看到输出的第一行是对应的输出数据列的格式，下面我将详细进行分析。

**Flags:**

- Flags为0则可计算得出该帧耗时：FrameCompleted - IntendedVsync。
- Flags为非0则表示绘制时间超过16ms，为异常帧。


**IntendedVsync：**

- 帧的预期Vsync时刻，如果预期的Vsync时刻与现实的Vsync时刻不一致，则表明UI线程中有耗时工作导致其无法响应Vsync信号。


**Vsync：**

- 花费在Vsync监听器和帧绘制的时间，比如Choreographer frame回调、动画、View.getDrawingTime等待。
- 理解Vsync：Vsync避免了在屏幕刷新时，把数据从后台缓冲区复制到帧缓冲区所消耗的时间。


**OldestInputEvent：**

- 输入队列中最旧输入事件的时间戳，如果没有输入事件，则此列数据都为Long.MAX_VALUE。
- 通常用于framework层开发。


**NewestInputEvent：**

- 输入队列中最新输入时间的时间戳，如果没有输入事件，则此列数据都为0。
- 计算App大致的延迟添加时间：FrameCompleted - NewestInputEvent。
- 通常用于framework层开发。


**HandleInputStart：**

- 将输入事件分发给App对应的时间戳时刻。
- 用于测量App处理输入事件的时间：AnimationStart - HandleInputStart。当值大于2ms时，说明程序花费了很长的时间来处理输入事件，比如View.onTouchEvent等事件。注意在Activity切换或产生点击事件时此值一般都比较大，此时是可以接受的。


**AnimationStart：**

- 运行Choreographer（舞蹈编排者）注册动画的时间戳。
- 用来评估所有运行的所有动画器（ObjectAnimator、ViewPropertyAnimator、常用转换器）需要多长时间：AnimationStart - PerformTraversalsStart。当值大于2ms时，请查看此时是否执行的是自定义动画且动画是否有耗时操作。


**PerformTraversalsStart：**

- 执行布局递归遍历开始的时间戳。
- 用于获取measure、layout的时间：DrawStart - PerformTraversalsStart。（注意滚动或动画期间此值应接近于0）。


**DrawStart：**

- draw阶段开始的时间戳，它记录了任何无效视图的DisplayList的起点。
- 用于获取视图数中所有无效视图调用View.draw方法所需的时间：SyncStart - DrawStart。
- 在此过程中，硬件加速模块中的DisplayList发挥了重要作用，Android系统仍然使用invalidate()调用draw()方法请求屏幕更新和渲染视图，但是对实际图形的处理方式有所不同。**Android系统并没有立即执行绘图命令，而是将它们记录在DisplayList中，该列表包含视图层次结构绘图所需的所有信息。相对于软件渲染的另一个优化是，Android系统仅需要记录和更新DispalyList，以显示被invalidate() 标记为dirty的视图。只需重新发布先前记录的Displaylist，即可重新绘制尚未失效的视图**。此时的硬件绘制模型主要包括三个过程：**刷新视图层级、记录和更新DisplayList、绘制DisplayList**。相对于软件绘制模型的刷新视图层级、然后直接去绘制视图层级的两个步骤，虽然多了一个步骤，但是节省了很多不必要的绘制开销。


**SyncQueued：**

- sync请求发送到RenderThread线程的时间戳。
- 获取sync就绪所花费的时间：SyncStart - SyncQueued。如果值大于0.1ms，则说明RenderThread正在忙于处理不同的帧。


**SyncStart：**

- 绘图的sync阶段开始的时间戳。
- IssueDrawCommandsStart - SyncStart > 0.4ms左右则表明有许多新的位图需要上传至GPU。


**IssueDrawCommandsStart：**

- 硬件渲染器开始GPU发出绘图命令的时间戳。
- 用于观察App此时绘制时消耗了多少GPU：FrameCompleted - IssueDrawCommandsStart。


**SwapBuffers：**

- eglSwapBuffers被调用时的时间戳。
- 通常用于Framework层开发。


**FrameCompleted：**

- 当前帧完成绘制的时间戳。
- 获取当前帧绘制的总时间：FrameCompleted - IntendedVsync。


综上，我们可以利用这些数据计算获取我们在自动化测试中想关注的因素，比如**帧耗时、该帧调用View.draw方法所消耗的时间**。framestats和帧耗时信息等一般2s收集一次，即一次120帧。为了精确控制收集数据的时间窗口，如将数据限制为特定的动画，可以重置计数器，重新聚合统计的信息，对应命令如下：

    adb shell dumpsys gfxinfo <PackageName> reset
    

### 2）、SurfaceFlinger

我们都知道，在Android 4.1以后，系统使用了三级缓冲机制，即此时有三个Graphic Buffer，那么**如何查看每个Graphic Buffer占用的内存呢？**

答案是使用SurfaceFlinger，命令如下所示：

    adb shell dumpsys SurfaceFlinger
    

输出的结果非常多，因为包含很多系统应用和界面的相关信息，这里我们仅过滤出Awesome-WanAndroid应用对应的信息：

    + Layer 0x7f5a92f000 (json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.MainActivity#0)
      layerStack=   0, z=    21050, pos=(0,0), size=(1080,2280), crop=(   0,   0,1080,2280), finalCrop=(   0,   0,  -1,  -1), isOpaque=1, invalidate=0, dataspace=(deprecated) sRGB Linear Full range, pixelformat=RGBA_8888 alpha=0.000, flags=0x00000002, tr=[1.00, 0.00][0.00, 1.00]
      client=0x7f5dc23600
      format= 1, activeBuffer=[1080x2280:1088,  1], queued-frames=0, mRefreshPending=0
            mTexName=386 mCurrentTexture=0
            mCurrentCrop=[0,0,0,0] mCurrentTransform=0
            mAbandoned=0
            - BufferQueue mMaxAcquiredBufferCount=1 mMaxDequeuedBufferCount=2
              mDequeueBufferCannotBlock=0 mAsyncMode=0
              default-size=[1080x2280] default-format=1 transform-hint=00 frame-counter=51
            FIFO(0):
            Slots:
              // 序号           // 表明是否使用的状态 // 对象地址 // 当前负责第几帧 // 手机屏幕分辨率大小
             >[00:0x7f5e05a5c0] state=ACQUIRED 0x7f5b1ca580 frame=51 [1080x2280:1088,  1]
              [02:0x7f5e05a860] state=FREE     0x7f5b1ca880 frame=49 [1080x2280:1088,  1]
              [01:0x7f5e05a780] state=FREE     0x7f5b052a00 frame=50 [1080x2280:1088,  1]


在Slots中，显示的是缓冲区相关的信息，可以看到，此时App使用的是00号缓冲区，即第一个缓冲区。

接着，在SurfaceFlinger命令输出log的最下方有一栏Allocated buffers，这这里可以使用当前缓冲区对应的对象地址去查询其占用的内存大小。具体对应到我们这里的是0x7f5b1ca580，匹配到的结果如下所示：

    0x7f5b052a00: 9690.00 KiB | 1080 (1088) x 2280 |    1 |        1 | 0x10000900 | json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.MainActivity#0
    0x7f5b1ca580: 9690.00 KiB | 1080 (1088) x 2280 |    1 |        1 | 0x10000900 | json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.MainActivity#0
    0x7f5b1ca880: 9690.00 KiB | 1080 (1088) x 2280 |    1 |        1 | 0x10000900 | json.chao.com.wanandroid/json.chao.com.wanandroid.ui.main.activity.MainActivity#0


可以看到，这里每一个Graphic Buffer都占用了9MB多的内存，通常分辨率越大，单个Graphic Buffer占用的内存就越多，如1080 x 1920的手机屏幕，一般占用8160kb的内存大小。此外，如果应用使用了其它的Surface，如SurfaceView或TextureView（两者一般用在opengl进行图像处理或视频处理的过程中），这个值会更大。如果当App退到后台，系统就会将这部分内存回收。

了解了常用布局优化常用的工具与命令之后，我们就应该开始着手进行优化了，但在开始之前，我们还得对Android的布局加载原理有比较深入的了解。

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f3c44ec306846e18e9225e189e51454~tplv-k3u1fbpfcp-zoom-1.image" width=30%>
</div>


# 四、总结（上）

在本篇文章中，我们主要对Android的布局绘制以及加载原理、优化工具、全局监控布局和控件的加载耗时进行了全面的讲解，这为大家学习《深入探索Android布局优化（下）》打下了良好的基础。下面，总结一下本篇文章涉及的五大主题：

- 1、绘制原理：CPU\GPU、Android图形系统的整体架构、绘制线程、刷新机制。
- 2、屏幕适配：OLED 屏幕和 LCD 屏幕的区别、屏幕适配方案。
- 3、优化工具：使用Systrace来进行布局优化、利用Layout Inspector来查看视图层级结构、采用Choreographer来获取FPS以及自动化测量 UI 渲染性能的方式（gfxinfo、SurfaceFlinger等dumpsys命令）。


下篇，我们将进入布局优化的原理分析环节，敬请期待~


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b91ceda8770b45719e155c84ddaad9a6~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、[国内Top团队大牛带你玩转Android性能分析与优化 第五章 布局优化](https://coding.imooc.com/class/308.html)

2、[极客时间之Android开发高手课 UI优化](https://time.geekbang.org/column/article/80921)

3、[手机屏幕的前世今生 可能比你想的还精彩](http://mobile.zol.com.cn/680/6809348.html)

4、[OLED 和 LCD 什么区别？](https://www.zhihu.com/question/22263252)

5、[Android 目前稳定高效的UI适配方案](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650826034&idx=1&sn=5e86768d7abc1850b057941cdd003927&chksm=80b7b1acb7c038ba8912b9a09f7e0d41eef13ec0cea19462e47c4e4fe6a08ab760fec864c777&scene=21#wechat_redirect)

6、[骚年你的屏幕适配方式该升级了!-smallestWidth 限定符适配方案](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650826381&idx=1&sn=5b71b7f1654b04a55fca25b0e90a4433&chksm=80b7b213b7c03b0598f6014bfa2f7de12e1f32ca9f7b7fc49a2cf0f96440e4a7897d45c788fb&scene=21#wechat_redirect)

7、[dimens_sw github](https://github.com/ladingwu/dimens_sw)

8、[一种极低成本的Android屏幕适配方式](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247484502&idx=2&sn=a60ea223de4171dd2022bc2c71e09351&scene=21#wechat_redirect)

9、[骚年你的屏幕适配方式该升级了!-今日头条适配方案](https://mp.weixin.qq.com/s/oSBUA7QKMWZURm1AHMyubA)

10、[今日头条屏幕适配方案终极版正式发布!](https://juejin.im/post/6844903697000972295#heading-0)

11、[使用Systrace分析UI性能](https://www.jianshu.com/p/b492140a555f)

12、[GAPID-Graphics API Debugger](https://gapid.dev/about/)

13、[Android性能优化之渲染篇](http://hukai.me/android-performance-render/)

14、[Android 屏幕绘制机制及硬件加速](https://blog.csdn.net/qian520ao/article/details/81144167)

15、[Android 图形处理官方教程](https://blog.csdn.net/qian520ao/article/details/81144167)

16、[Vulkan - 高性能渲染](https://zhuanlan.zhihu.com/p/20712354)

17、[Android Vulkan Tutorial](https://source.android.com/devices/graphics/arch-vulkan)

18、[Test UI performance-gfxinfo](https://developer.android.com/training/testing/performance#top_of_page)

19、[使用dumpsys gfxinfo 测UI性能（适用于Android6.0以后）](https://www.jianshu.com/p/7477e381a7ea)

20、[TextureView API](https://developer.android.com/reference/android/view/TextureView)

21、[PrecomputedText API](https://developer.android.com/reference/android/text/PrecomputedText)

22、[Litho Tutorial](https://fblitho.com/docs/tutorial)

23、[基本功 | Litho的使用及原理剖析](https://tech.meituan.com/2019/03/14/litho-use-and-principle-analysis.html)

24、[Flutter官方文档中文版](https://flutter.cn/docs/resources/technical-overview)

25、[[Google Flutter 团队出品] 深入了解 Flutter 的高性能图形渲染](https://www.bilibili.com/video/av48772383/?spm_id_from=333.788.videocard.0)

26、[Flutter渲染机制—UI线程](http://gityuan.com/2019/06/15/flutter_ui_draw/)

27、[RenderThread:异步渲染动画](https://mp.weixin.qq.com/s?__biz=MzUyMDAxMjQ3Ng==&mid=2247489230&amp;idx=1&amp;sn=adc193e35903ab90a4c966059933a35a&source=41#wechat_redirect)

28、[RenderScript官方文档](https://developer.android.com/guide/topics/renderscript/compute)

29、[RenderScript :简单而快速的图像处理](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0504/4205.html?utm_source=itdadao&utm_medium=referral)

30、[RenderScript渲染利器](https://www.jianshu.com/p/b72da42e1463)


## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ac9e50387674040860cdfd50d67fa80~tplv-k3u1fbpfcp-zoom-1.image" width=20%><img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02cfec0590ce4466b980e3ab72334651~tplv-k3u1fbpfcp-zoom-1.image" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/415f15ac5d784f5d974f3ca9abe9d05c~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。