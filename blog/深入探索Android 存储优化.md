---

		title:  深入探索Android 存储优化
		date: 2019/12/29 22:12:00   
		tags: 
		- 性能优化
		categories: 性能优化
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

本篇是Androi绘制优化的进阶篇，难度会比较大，建议对内存优化不是非常熟悉的前仔细看看在前几篇文章中，笔者曾经写过的一篇[Android性能优化之内存优化](https://jsonchao.github.io/2019/08/18/Android%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E4%B9%8B%E5%86%85%E5%AD%98%E4%BC%98%E5%8C%96/)，其中详细分析了以下几大模块：

- Android的内存管理机制
- 优化内存的意义
- 避免内存泄漏
- 优化内存空间
- 图片管理模块的设计与实现

如果你对以上基础内容都比较了解了，那么我们便开始接下来的Android内存优化探索之旅吧。





















##### 参考链接：
---
1、[国内Top团队大牛带你玩转Android性能分析与优化 第四章 内存优化](https://coding.imooc.com/class/308.html)

2、[极客时间之Android开发高手课 内存优化](https://time.geekbang.org/column/article/71277)

3、[微信 Android 终端内存优化实践](https://mp.weixin.qq.com/s/KtGfi5th-4YHOZsEmTOsjg?)

4、[GMTC－Android内存泄漏自动化链路分析组件Probe.key](https://static001.geekbang.org/con/19/pdf/593bc30c21689.pdf)

5、[Manage your app's memory](https://developer.android.com/topic/performance/memory#monitor)

6、[Overview of memory management](https://developer.android.com/topic/performance/memory-overview.html)

7、[Android内存优化杂谈](https://mp.weixin.qq.com/s/Z7oMv0IgKWNkhLon_hFakg)

8、[Android性能优化之内存篇](http://hukai.me/android-performance-memory/)

9、[管理应用的内存](http://hukai.me/android-training-managing_your_app_memory/)

10、《Android移动性能实战》第二章 内存


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
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/wexin_qrcode.jpg" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。