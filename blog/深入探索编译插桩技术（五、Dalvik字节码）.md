---

		title:  深入探索编译插桩技术（五、Dalvik字节码）
		date: 2020/2/5 17:43:00   
		tags: 
		- Android进阶
		categories: Android进阶
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。



# Java字节码与Dalvik字节码的区别

不同于Java虚拟机运行的是Class文件，它的内部对应的是Java字节码。在Android这种嵌入式平台上，为了提升性能，Android虚拟机运行的是Dex文件，Google专门为其设计了一种Dalvik字节码，它以增加指令长度的方式缩减了指令的数量，使指令指令的执行效率更高。下面，我们就以一个实例来看看Java字节码与Dalvik字节码的差别。

首先，准备一个JsonChao.java文件，其内容如下所示：





然后，通过javac与javap命令查看对应的Java字节码，命令如下：

    
    // 生成JsonChao.class，也就是Java字节码
    javac JsonChao.java   
    // 查看JsonChao类的Java字节码
    javap -v JsonChao   
    
    
生成的JsonChao.class文件内容如下所示：




然后，通过dx与dexdump命令查看对应的Dalvik字节码，命令如下：


    //通过Java字节码，生成Dalvik字节码
    dx --dex --output=JsonChao.dex JsonChao.class  
    // 查看Sample.dex的Dalvik的字节码
    dexdump -d JsonChao.dex   
    
    
生成的JsonChao.dex文件内容如下所示：



可以看到，Java字节码和Dalivk字节码的格式与指令形式都有比较大的差别，其主要区别有如下三种：

## 1、体系结构的差异

相对于JVM基于栈的实现而言，Dalvik VM主要是基于寄存器而实现的。并且，在ARM平台下，寄存器的实现其性能会高于栈实现。两者的体系结构对比如下图所示：

![image](https://user-gold-cdn.xitu.io/2020/4/3/1713def99f72e45d?w=0&h=0&f=ico&s=4286)


### 2、不同的格式结构

在Class文件中，每个文件都会有其对应的常量池和其它一些公共字段。而在Dex文件中，每个一个Dex文件中的Class都会共用同一个常量池和公共字段。所以Dex的格式结构比Class文件要更加紧凑，因而Dex文件的体积相对来说要更小。


### 3、字节码指令层次的深度优化

相对于Java字节码，Dalvik字节码在字节码指令的层次上做了深度优化，从而变得更加精简，在实现相同的代码Dalvik字节码只需要几条指令的情况下，使用Java字节码实现很可能需要上百条指令。如下图所示：




[官方文档-Dex文件格式](https://source.android.com/devices/tech/dalvik/dex-format)

[官方文档-Dalvik字节码](https://source.android.com/devices/tech/dalvik/dalvik-bytecode)


Dalvik and ART PDF

Understanding the Davlik Virtual Machine PDF



















##### 参考链接：
---
1、[国内Top团队大牛带你玩转Android性能分析与优化 第6章 卡顿优化](https://coding.imooc.com/class/308.html)

2、[极客时间之Android开发高手课 卡顿优化](https://time.geekbang.org/column/article/71982)

3、《Android移动性能实战》第四章 CPU

4、《Android移动性能实战》第七章 流畅度

5、[Android dumpsys cpuinfo 信息解读](https://blog.csdn.net/lindroid/article/details/90904947)

6、[如何清楚易懂的解释“UV和PV＂的定义？](https://www.zhihu.com/question/20448467)

7、[nanoscope-An extremely accurate Android method tracing tool](https://github.com/uber/nanoscope)

8、[DroidAssist-A lightweight Android Studio gradle plugin based on Javassist for editing bytecode in Android.](https://github.com/didi/DroidAssist)

9、[lancet-A lightweight and fast AOP framework for Android App and SDK developers](https://github.com/eleme/lancet)

10、[MethodTraceMan-用于快速找到高耗时方法，定位解决Android App卡顿问题](https://github.com/zhengcx/MethodTraceMan)

11、[Linux环境下进程的CPU占用率](http://www.samirchen.com/linux-cpu-performance/)

12、[使用 ftrace](https://source.android.com/devices/tech/debug/ftrace?hl=zh-cn)

13、[profilo-A library for performance traces from production](https://github.com/facebookincubator/profilo)

14、[ftrace 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-ftrace/index.html)

15、[atrace源码](https://android.googlesource.com/platform/frameworks/native/+/master/cmds/atrace/atrace.cpp)

16、[MCI：移动持续集成在大众点评的实践](https://tech.meituan.com/2018/07/12/mci.html)

17、[aapt2官方教程](https://developer.android.com/studio/command-line/aapt2)


## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/wexin_play.jpg" width=20%><img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/Apaliy.jpg" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Performance/master/screenshots/Awesome-Android.png" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。