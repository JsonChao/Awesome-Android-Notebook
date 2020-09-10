---

		title:  深入探索Android启动速度优化（下）
		date: 2020/2/11 22:02:00   
		tags: 
		- 性能优化
		categories: 性能优化
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


在性能优化的整个知识体系中，最重要的就是稳定性优化，在上一篇文章 [《深入探索Android稳定性优化》](https://juejin.im/post/6844903972587716621) 中我们已经深入探索了Android稳定性优化的疆域。那么，**除了稳定性以外，对于性能纬度来说，哪个方面的性能是最重要的呢**？毫无疑问，就是**应用的启动速度**。下面，**就让我们扬起航帆，一起来逐步深入探索Android启动速度优化的奥秘**。


# 思维导图大纲

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ba37d61c8ae439fb27cd4723a576816~tplv-k3u1fbpfcp-zoom-1.image)


# 目录

- 五、[启动优化黑科技](https://juejin.im/post/6844904093786308622#heading-167)
    - 1、启动阶段抑制GC
    - 2、CPU锁频
    - 3、IO优化
    - 4、数据重排
    - 5、类加载优化（Dalvik）
    - 6、保活
- 六、[启动优化的常见问题](https://juejin.im/post/6844904093786308622#heading-254)
    - 1、启动优化是怎么做的？
    - 2、是怎么异步的，异步遇到问题没有？
    - 3、启动优化有哪些容易忽略的注意点？
    - 4、版本迭代导致的启动变慢有好的解决方式吗？
- 七、[总结](https://juejin.im/post/6844904093786308622#heading-259)
    - 1、优化总方针
    - 2、注意事项


# 五、启动优化黑科技

## 1、启动阶段抑制GC

**启动时CG抑制，允许堆一直增长，直到手动或OOM停止GC抑制**。（空间换时间）


### 前提条件

- 1、设备厂商没有加密内存中的Dalvik库文件。
- 2、设备厂商没有改动Google的Dalvik源码。


### 实现原理

- 1、首先，**在源码级别找到抑制GC的修改方法，例如改变跳转分支**。
- 2、然后，**在二进制代码里找到 A 分支条件跳转的"指令指纹"，以及用于改变分支的二进制代码，假设为 override_A**。
- 3、最后，**应用启动后扫描内存中的 libdvm.so，根据"指令指纹"定位到修改位置，并使用 override_A 覆盖**。


### 缺点

需要**白名单覆盖**所有设备，但维护成本高。


## 2、CPU锁频

一个设备的CPU通常都是4核或者8核，但是应用在一般情况下对CPU的利用率并不高，可能只有30%或者50%，如果我们在启动速度暴力拉伸CPU频率，以此提高CPU的利用率，那么，应用的启动速度会提升不少。

在Android系统中，**CPU相关的信息存储在/sys/devices/system/cpu目录的文件中，通过对该目录下的特定文件进行写值，实现对CPU频率等状态信息的更改**。


### 缺点

暴力拉伸CPU频率，导致耗电量增加。


### CPU工作模式

- **performance**：**最高性能模式**，即使系统负载非常低，cpu也在**最高频率**下运行。
- **powersave**：**省电模式**，与performance模式相反，cpu始终在**最低频率**下运行。
- **ondemand**：CPU频率跟**随系统负载**进行**变化**。
- **userspace**：可以简单理解为**自定义模式**，在该模式下可以**对频率进行设定**。


### CPU的工作频率范围

对应的文件有：

- cpuinfo_max_freq
- cpuinfo_min_freq
- scaling_max_freq
- scaling_min_freq


## 3、IO优化

- 1、启动过程**不建议出现网络IO**。
- 2、为了**只解析启动过程中用到的数据**，应**选择合适的数据结构**，如将ArrayMap改造成支持随机读写、延时解析的数据存储结构以替代SharePreference。


这里需要注意的是，需要**考虑重度用户的使用场景**。


### 补充加油站：Linux IO知识

#### 1、磁盘高速缓存技术

利用内存中的存储空间来暂存从磁盘中读出的一系列盘块中的信息。因此，磁盘高速缓存在逻辑上属于磁盘，物理上则是驻留在内存中的盘块。

其内存中分为两种形式：

- 在内存中开辟一个单独的存储空间作为磁速缓存，大小固定。
- 把未利用的内存空间作为一个缓沖池，供请求分页系统和磁盘I/O时共享。


#### 2、分页

- 存储器管理的一种技术。
- 可以使电脑的主存使用存储在辅助存储器中的数据。
- 操作系统会将辅助存储器（通常是磁盘）中的数据分区成固定大小的区块，称为“页”（pages）。
当不需要时，将分页由主存（通常是内存）移到辅助存储器；当需要时，再将数据取回，加载主存中。
- 相对于分段，分页允许存储器存储于不连续的区块以维持文件系统的整齐。
- 分页是磁盘和内存间传输数据块的最小单位。


#### 3、高速缓存/缓冲器

- 都是介于高速设备和低速设备之间。
- 高速缓存存放的是低速设备中某些数据的复制数据，而缓冲器则可同时存储高低速设备之间的数据。
- 高速缓存存放的是高速设备经常要访问的数据。


#### 4、linux同步IO：sync、fsync、msync、fdatasync

##### 为什么要使用同步IO？

当数据写入文件时，内核通常先将该数据复制到缓冲区高速缓存或页面缓存中，如果该缓冲区尚未写满，则不会将其排入输入队列，而是等待其写满或内核需要重用该缓冲区以便存放其他磁盘块数据时，再将该缓冲排入输出队列，最后等待其到达队首时，才进行实际的IO操作—延迟写。

延迟写减少了磁盘读写次数，但是却降低了文件内容的更新速度，可能会造成文件更新内容的丢失。为了保证数据一致性，则需使用同步IO。


##### sync

- sync函数只是将所有修改过的块缓冲区排入写队列，然后就返回，它并不等待实际磁盘写操作结束再返回。
- 通常称为update的系统守护进程会周期性地（一般每隔30秒）调用sync函数。这就保证了定期冲洗内核的块缓冲区。


##### fsync

- fsync函数只对文件描述符filedes指定的单一文件起作用，并且等待磁盘IO写结束后再返回。通常应用于需要确保将修改内容立即写到磁盘的应用如数据库。
- 文件的数据和metadata通常存放在硬盘的不同地方，因此fsync至少需要两次IO操作。


##### msync

如果当前硬盘的平均寻道时间是3-15ms，7200RPM硬盘的平均旋转延迟大约为4ms，因此一次IO操作的耗时大约为10ms。

如果使用内存映射文件的方式进行文件IO（mmap），将文件的page cache直接映射到进程的地址空间，这时需要使用msync系统调用确保修改的内容完全同步到硬盘之上。


##### fdatasync

- fdatasync函数类似于fsync，但它只影响文件的数据部分。而fsync还会同步更新文件的属性。
- 仅仅只在必要（如文件尺寸需要立即同步）的情况下才会同步metadata，因此可以减少一次IO操作。


##### 日志文件都是追加性的，文件尺寸一致在增大，如何利用好fdatasync减少日志文件的同步开销？

创建每个log文件时先写文件的最后一个page，将log文件扩展为10MB大小，这样便可以使用fdatasync，每写10MB只有一次同步metadata的开销。


#### 5、磁盘IO与网络IO

##### 磁盘IO（缓存IO）

标准IO，大多数文件系统默认的IO操作。

- 数据先从磁盘复制到内核空间的缓冲区，然后再从内核空间中的缓冲区复制到应用程序的缓冲区。
- 读操作：操作系统检查内核的缓冲区有没有需要的数据，如果已经有缓存了，那么直接从缓存中返回；否则，从磁盘中返回，再缓存在操作系统的磁盘中。
- 写操作：将数据从用户空间复制到内核空间中的缓冲区中，这时对用户来说写操作就已经完成，至于什么时候写到磁盘中，由操作系统决定，除非显示地调用了sync同步命令。


**优点**

- 在一定程度上分离了内核空间和用户空间，保护系统本身安全。
- 可以减少磁盘IO的读写次数，从而提高性能。

**缺点**

DMA方式可以将数据直接从磁盘读到页缓存中，或者将数据从页缓存中写回到磁盘，而不能在应用程序地址空间和磁盘之间进行数据传输，这样，数据在传输过程中需要在应用程序地址空间（用户空间）和缓存（内核空间）中进行多次数据拷贝操作，这带来的CPU以及内存开销是非常大的。

**磁盘IO主要的延时（15000RPM硬盘为例）**

机械转动延时（平均2ms）+ 寻址延时（2~3ms）+ 块传输延时（0.1ms左右）=> 平均5ms

**网络IO主要延时**

服务器响应延时 + 带宽限制 + 网络延时 + 跳转路由延时 + 本地接收延时（一般为几十毫秒到几千毫秒，受环境影响极大）


#### 6、PIO与DMA

##### PIO

很早之前，磁盘和内存之间的数据传输是需要CPU控制的，也就是读取磁盘文件到内存中时，数据会经过CPU存储转发，这种方式称为PIO。


##### DMA（直接内存访问，Direct Memory Access）

- 可以不经过CPU而直接进行磁盘和内存的数据交换。
- CPU只需要向DMA控制器下达指令，让DMA控制器来处理数据的传送即可。
- DMA控制器通过系统总线来传输数据，传送完毕再通知CPU，这样就在很大程度上降低了CPU占用率，大大节省了系统资源，而它的传输速度与PIO的差异并不明显，而这主要取决于慢速设备的速度。


#### 7、直接IO与异步IO

##### 直接IO

应用程序直接访问磁盘数据，而不经过内核缓冲区。以减少从内核缓冲区到用户数据缓存的数据复制。


##### 异步IO

当访问数据的线程发出请求后，线程会接着去处理其它事情，而不是阻塞等待。


#### 8、VFS（虚拟文件系统，Virtual File System）

可以为访问文件系统的系统调用提供一个统一的抽象接口。


## 4、数据重排

Dex文件用到的类和APK里面各种资源文件都比较小，读取频繁，且磁盘地址分布范围比较广。我们可以利用Linux文件IO流程中的page cache机制将它们按照读取顺序重新排列在一起，以减少真实的磁盘IO次数。


### 1、类重排

使用Facebook的 [ReDex](https://github.com/facebook/redex) 的Interdex调整类在Dex中的排列顺序。


### 2、资源文件重排

- 1、最佳方案是修改内核源码，实现统计、度量、自动化，其次也可以使用Hook框架进行统计得出资源加载顺序列表。
- 2、最后，调整apk文件列表需要修改7zip源码以支持传入文件列表顺序。


## 技术视野

- 所谓的创新，不一定是要创造前所未有的东西，也可以将已有的方案移植到新的平台，并结合该平台的特性落地，就是一个很大的创新。
- 当我们足够熟悉底层的知识时，可以利用系统的特性去做更加深层次的优化。


### 3、了解Hook框架

#### Xposed框架是什么？

一个可以不修改APK就影响程序运行的Hook框架。


#### 原理

用自身实现的app_process替换掉系统/system/bin/app_process，加载一个额外的XposedBridge的jar包，用于将入口osZygoteInit.main()替换成XposedBridge.main()。之后，创建的Zygote进程和其子进程都是Hook过的了。

使用具体细节参见[Xposed教程](https://blog.csdn.net/coder_pig/article/details/80031285)。


## 5、类加载优化（Dalvik）

### 1、类预加载原理

对象第一次创建的时候，JVM首先检查对应的Class对象是否已经加载。如果没有加载，JVM会根据类名查找.class文件，将其Class对象载入。同一个类第二次new的时候就不需要加载类对象，而是直接实例化，创建时间就缩短了。


### 2、类加载优化过程

- 在Dalvik VM加载类的时候会有一个类校验过程，它需要校验方法的每一个指令。
- 通过Hook去掉verify步骤 -> 几十ms的优化
- 最大优化场景在于首次安装和覆盖安装时，在Dalvik平台上，一个2MB的Dex正常需要350ms，将classVerifyMode设为VERIFY_MODE_NONE后，只需150ms，节省超过50%时间。

ART比较复杂，Hook需要兼容几个版本。而且在安装时，大部分Dex已经优化好了，去掉ART平台的verify只会对动态加载的Dex带来一些好处。所以暂时不建议在ART平台使用。


### 3、延伸：插件化和热修复

它们在设计上都存在大量的Hook和私有API调用，共同的缺点有如下两类问题。


##### 1、稳定性较差

由于厂商的兼容性、安装失败、ART加载时dex2oat失败等原因，还是会有一些代码和资源的异常。Android P推出的non-sdk-interface调用限制，以后适配只会越来越难，成本越来越高。


##### 2、性能问题

用到一些黑科技导致底层Runtime的优化享受不到。如Tinker加载补丁后，启动速度会降低5%~10%。


#### 1、各项热补丁技术的优缺点

##### 缺点

- 只针对单一客户端版本，随着版本差异变大补丁体积也会变大。
- 不支持所有修改，如AndroidManifest。
- 对代码和资源的更新成功率无法达到100%。


##### 优点

- 降低开发成本，轻量而快速地升级。发布补丁等同于发布版本，也应该完整地执行测试与上线流程。
- 远端调试，只为特定用户发送补丁。
- 数据统计，对同一批用户更换补丁版本，能够更好地进行ABTest，得到更精确的数据。


#### 2、InstanceRun实现机制

Android官方使用热补丁技术实现InstantRun。


##### 应用构建流程

构建 -> 部署 -> 安装 -> 重启app -> 重启activity


##### 实现目标

尽可能多的剔除不必要的步骤，然后提升必要步骤的速度。


##### InstantRun构建的三种方式

**1、HotSwap**

增量构建 -> 改变部署


**场景：**

适用于多数简单的改变（包括一些方法实现的修改，或者变量值修改）。


**2、Warm Swap**

增量构建 -> 改变部署 -> activity重启


**场景：**

一般是修改了resources。


**3、Cold Swap**

增量构建 -> 改变部署 -> 应用重启 -> activity重启


**场景：**

涉及结构性变化，如修改了继承规则或方法签名。


##### 首次运行Instant Run，Gradle执行的操作

- 在有Instant Run的环境下：一个新的App Server会被注入到App中，与Bytecode instrumentation协同监控代码的变化。
- 同时会有一个新的Application类，它注入了一个自定义类加载器。同时该Application会启动我们所需的新注入的App Server。于是，AndroidManifest会被修改来确保我们能使用这个新的Application。
- 使用的时候，它会通过决策，合理运用冷温热拔插来协助我们大量地缩短构建程序的时间。


##### HotSwap原理

Android Studio monitors
运行着Gradle任务来生成增量.dex文件（dex对应着开发中的修改类），AS会提取这些.dex文件发送到App Server，然后部署到App。因为原来版本的类都装载在运行中的程序了，Gradle会解释更新好这些.dex文件，发送到App Server的时候，交给自定义的类加载器来加载.dex文件。
App Server会不断地监听是否需要重写类文件，如果需要，任务会被立马执行，新的更改便能立即被响应。

需要注意的是，此时InstantRun是不能回退的，必须重启应用响应修改。


##### WarmSwap原理

因为资源文件是在Activity创建时加载，所以必须重启Activity加载资源文件。

注意：AndroidManifest的值是在APK安装的时候被读取的，所以需要触发一个完整的应用构建和部署。


##### ColdSwap原理

应用部署的时候，会把工程拆分成十个部分，每个部分都拥有自己的.dex文件，然后所有的类会根据包名被分配给相应的.dex文件。当ColdSwap开启时，修改过的类所对应的的.dex文件，会重组生成新的.dex文件，然后再部署到设备上。


注意：应用多进程会被降级为ColdSwap。


#### 3、apk打包流程

manifest文件合并、打包，和res一起被AAPT合并到APK中，同时项目代码被编译成字节码，然后转换成.dex文件，也被合并到APK中。

##### Android打包流程回顾，最后对于release签名apk需要进行zipalign优化，它是指什么？

在回答这个问题之前，我们需要先了解下内存对齐（DSA，Data Structure Alignment）：

    各种类型的数据按照一定的规则在内存空间上排列，这就是对齐。


**内存对齐的优势在于能够以空间换时间，减少数据存取指令周期，提升程序运行时的速度**。


##### 编译器内存字节对齐的原则是什么？

- 1、数据类型的自身对齐值就是其长度（64位 OS）。
- 2、结构体或类的自身对齐值就是成员中自身对齐值最大的那个。需要起始地址必须是其相应有效对齐值的整数，并要求结构体的大小也为该结构体有效对齐值的整数倍。


zipalign优化的最根本目的是**帮助操作系统更高效地根据请求索引资源，使用resource-handling code统一将DSA限定为4byte。**


##### 手动执行Align优化

利用build-tools文件夹下对应Android版本中的zipalign工具：

    zipalign -v 4 source.apk androidres.apk
 
   
检查当前APK是否已经执行过Align优化：

    zipalign -c -v 4 androidres.apk
  
   
其中：
 
- -c：检查。
- -v：代表详细输出。
- 4：代表对齐为4个字节。


#### 4、AndFix

##### 实现原理

native hook -> dalvik_repleaceMethod -> 无法支持新增或删除filed的情况 -> 需修复特定问题


##### 优点

- 立即生效
- 补丁较小


##### 缺点

- 兼容性不佳
- 开发不透明


#### 5、Qzone

它是一个基于Android Dex分包方案。它将多个dex文件放入到app的classloader中，但是android dex拆包方案中的类是没有重复的，如果classes.dex和classes1.dex中有重复的类，**当用到这个重复的类时，系统会选择哪个类进行加载呢？**

一个ClassLoader可以包含多个dex文件，每个dex文件是一个Elements，多个dex文件排列成有序的dexElements，当找类的时候，会按顺序遍历dex文件，然后从当前遍历的dex文件中找类，如果找到则返回，如果找不到从下一个dex文件继续查找。

所以，如果在不同的dex中有相同的类存在，那么会优先选择排在前面的dex文件的类。

Qzone热补丁方案就是把有问题的类打包到一个dex（patch.dex）中去，然后把这个dex插入到Elements的最前面。


##### 实现中遇到的问题

**1、当其它dex文件中的类引用了patch.dex中的类时，会出现校验错误。拆分dex的很多类都不是在同一个dex内的，怎么没有问题？**

因为这个校验有个前提，当引用类被打上了CLASS_ISPREVERIFIED标志，那么就会进行dex的校验。


**2、CLASS_ISPREVERIFIED标志是什么时候被打上去的？**

- 在dex转换成odex（dexopt过程）时，当apk在安装的时候，apk中的classes.dex会被虚拟机（dexopt）优化成odex文件，然后才会拿去执行。
- 虚拟机在启动的时候，会有许多的启动参数，其中一项就是verify选项，当verify选项被打开时，doVerify变量为true，那么就会执行dvmVerifyClass进行类的校验，如果校验成功，这个类会被打上CLASS_ISPREVERIFIED标志。


##### 具体的校验过程是怎么样的？

有两步验证：

1、验证clazz -> directMethods方法，其包含以下方法：

- static方法
- private方法
- 构造函数


2、clazz -> virtualMethods

- 虚函数 = override方法


如果以上方法中直接引用到的类（第一层级关系，不会进行递归搜索）和clazz**都在同一个dex中**的话，那么这个类就会被打上CLASS_ISPREVERIFED标志。

为了解决补丁方案中遇到的问题，所以必须从这些方法中入手，防止类被打上CLASS_ISPREVERIFIED标志。空间的方案是往所有类的构造函数里面插入一段代码：


    If (ClassVerifier.PREVENT_VERIFY) {
        System.out.println(AntilazyLoad.class);
    }
    
    
其中AntilazyLoad类会被打包成单独的hack.dex，这样当安装apk的时候，classes.dex中的类都会引用一个在不同dex中的AntilazyLoad类，这样就防止类被打上了CLASS_ISPREVERIFILED标志，只要没被打上这个标志的类都可以进行打补丁操作。


**注意：**

- 1、在应用启动进行加载时，AntilazyLoad类所在的dex包必须先加载进来，不然AntilazyLoad类会被标记为不存在，即使后续加载了hack.dex包，那么它也是不存在的。
- 2、当在Application的onCreate中加载hack.dex时，Application不能插入上述代码。


**为什么要选择构造函数？**

因为他不增加方法数，一个类即使没有显示的构造函数，也有一个隐式的默认构造函数。


##### 如何更高效地插入上述代码？

可以使用ASM/javaassist库在编译期间将相应的字节码插入Class文件中。


##### Art的处理

Art采用了新的方式，插桩对代码的执行效率没有影响。但是补丁中的类出现修改类变量或者方法，可能会导致出现内存地址错乱的情况。


**原因：**

dex2oat时fast*已经将类能确定的各个地址写死。如果运行时补丁包的地址出现改变，原始类去调用时就会出现地址错乱。


**解决方法：**

将其父类以及调用类的所有类都加入到补丁包中。


##### 虚拟机在安装期间为类打上CLASS_ISPREVERIFIED标志是为了什么？

为了提高性能。


##### 禁用CLASS_ISPREVERIFIED是否会影响APP的性能？

由于现在很多App都使用了MultiDex分包方案，这导致了很多类都没有被打上这个标志，所以此时禁用所有类打上CLASS_ISPREVERIFIED标志对性能的影响不是很大。


##### 如何有效地生成补丁包？

- 1、在正式版本发布的时候，会生成一份缓存文件，里面记录了所有class文件的MD5值，还有一份mapping混淆文件。
- 2、在后续的版本中使用-applaymapping选项，应用正式版本的mapping文件，然后计算编译完成的class文件的MD5和正式版本进行比较，把不相同的class文件打包成补丁包。


##### Qzone方案缺点

在补丁包大小与性能损耗上有一定的局限性。


#### 6、ASM字节码插桩

插桩就是将一段代码插入或者替换原本的代码。
字节码插桩就是在我们的代码编译成字节码（Class）后，在Android下生成dex之前修改Class文件，修改或者增强原有代码逻辑的操作。

除了AspectJ、Javassist框架外，还有一个应用更为广泛的ASM框架同样也是字节码操作框架，Instant Run包括Javassist就是借助ASM来实现各自的功能。

可以这样理解Class字节码与ASM之间的联系：

    JSON对于GSON就类似于字节码Class对于Javassist/ASM。


##### Android ASM自动埋点方案实践

Android 1.5.0版本以后提供了Transform API，允许第三方Plugin在打包dex文件之前的编译过程中操作.class文件，我们做的就是实现Transform进行.class文件遍历拿到所有方法，修改完成后对文件进行替换。

大致的流程如下所示：

**1、自动埋点追踪，遍历所有文件更换字节码**

    AutoTransform -> transform -> inputs.each {TransformInput input -> input.jarInput.each { JarInput jarInput -> … } input.directoryInputs.each { DirectoryInput directoryInput -> … }}


**2、Gradle插件实现**

    PluginEntry -> apply -> def android = project.extensions.getByType(AppExtension)

    registerTransform(android) -> AutoTransform transform = new AutoTransform

    android.registerTransform(transform)


**3、使用ASM进行字节码编写**

**ASM框架核心类**

- ClassReader：读取编译后的.class文件。
- ClassWriter：重新构建编译后的类。
- ClassVisitor：拜访类成员信息。
- AdviceAdapter：实现MethodVisitor接口，拜访方法的信息。

1、visit -> 在ClassVisitor中根据判断是否是实现View$OnClickListener接口的类，只有满足条件的类才会遍历其中的方法进行操作。

2、在MethodVisitor中对该方法进行修改


    visitAnnotation -> onMethodEnter -> onMethodExit


3、先在java文件中编写要插入的代码，然后使用ASM插件查看对应的字节码，根据其用ASM提供的Api一一对应地把代码填进来即可。

关于编译插桩的知识，笔者后面会有一系列的文章进行深入讲解，具体的文章目录可以在[这里查看](https://github.com/JsonChao/Awesome-Android-Architecture#%E7%BC%96%E8%AF%91%E6%8F%92%E6%A1%A9%E6%8A%80%E6%9C%AF%E8%BF%9B%E8%A1%8C%E4%B8%AD)。


#### 7、Tinker

##### 原理

- 全量替换新的Dex
- 在编译时通过新旧两个Dex生成差异patch.dex。在运行时，将差异patch.dex重新跟原始安装包的旧Dex还原为新的Dex。由于比较耗费时间与内存，放在后台进程:patch中，为了补丁包尽可能小，微信自研了DexDiff算法，它深度利用Dex的格式来减少差异的大小。


DexDiff的粒度是Dex格式的每一项，BsDiff的粒度是文件，AndFix/Qzone的粒度为class。


##### 缺点

- 1、占用Rom体积，1.5倍所修改Dex大小 = Dex.jar + dexopt文件。
- 2、一个额外的合成过程，合成时间长短和额外的内存消耗也会影响最终的成功率。


##### 热补丁方案对比

若不care性能损耗与补丁包大小，Qzone是最简单且成功率最高的方案。


#### 8、完善的热补丁系统构建

##### 一、网络通道

负责将补丁包交付给用户，包括特定用户和全量用户。

**1、pull通道**

在登录/24小时等时机，通过pull方式查询后台是否有对应的补丁包更新。


**2、指定版本的push通道**

在紧急情况下，我们可以在一个小时内向所有用户下发补丁包更新。


**3、指定特定用户的push通道**

对特定用户或用户组做远程调试。


##### 二、上线与管理平台

快速上线，管理历史记录，以及监控补丁的运行情况。


## 6、保活

### 1、厂商合作

### 2、微信Hardcoder

构建了App与系统（ROM）之间可靠的通信框架，让系统知道App的需求。


#### 原理

- 1、其实质是让**App跨过Framework直接跟厂商ROM通信**。
- 2、分为Client端和Server端，**Server端由厂商系统侧自行实现**。
- 3、它们直接采用 **LocalSocket** 方式，Hardcoder是 **Native** 实现的，使用了**Linux的Socket接口**实现了一套自己的LocalSocket。


#### 性能提升有多少？

平均10%~30%。


### 3、OPPO Hyper Boost加速引擎

一种**优化资源调度**的技术。


#### 原理

让应用程序与系统资源实现实时"双向对话"。当来自应用和游戏程序的**不同场景和用户行为**被Hyper Boost识别后，手机会**智能地匹配到合理的系统资源，让手机SoC的CPU、GPU、ISP、DSP提供的运算资源更加合理地利用**，从而让用户使用手机**更加流畅**。


# 六、启动优化的常见问题

## 1、启动优化是怎么做的？

- **1、分析现状、确认问题**
- **2、针对性优化（先概括，引导其深入）**
- **3、长期保持优化效果**


在某一个版本之后呢，我们会发现这个启动速度变得特别慢，同时用户给我们的反馈也越来越多，所以，我们开始考虑对应用的启动速度来进行优化。然后，我们就对启动的代码进行了代码层面的梳理，我们发现应用的启动流程已经非常复杂，接着，我们通过一系列的工具来确认是否在主线程中执行了太多的耗时操作。

我们经过了细查代码之后，发现应用主线程中的任务太多，我们就想了一个方案去针对性地解决，也就是进行异步初始化。（引导=>第2题） 然后，我们还发现了另外一个问题，也可以进行针对性的优化，就是在我们的初始化代码当中有些的优先级并不是那么高，它可以不放在Application的onCreate中执行，而完全可以放在之后延迟执行的，因为我们对这些代码进行了延迟初始化，最后，我们还结合了idealHandler做了一个更优的延迟初始化的方案，利用它可以在主线程的空闲时间进行初始化，以减少启动耗时导致的卡顿现象。做完这些之后，我们的启动速度就变得很快了。

最后，我简单说下我们是怎么长期来保持启动优化的效果的。首先，我们做了我们的启动器，并且结合了我们的CI，在线上加上了很多方面的监控。（引导=> 第4题）


## 2、是怎么异步的，异步遇到问题没有？

- **1、体现演进过程**
- **2、详细介绍启动器**


我们最初是采用的普通的一个异步的方案，即new Thread + 设置线程优先级为后台线程的方式在Application的onCreate方法中进行异步初始化，后来，我们使用了线程池、IntentService的方式，但是，在我们应用的演进过程当中，发现代码会变得不够优雅，并且有些场景非常不好处理，比如说多个初始化任务直接的依赖关系，比如说某一个初始化任务需要在某一个特定的生命周期中初始化完成，这些都是使用线程池、IntentService无法实现的。所以说，我们就开始思考一个新的解决方案，它能够完美地解决我们刚刚所遇到的这些问题。

这个方案就是我们目前所使用的启动器，在启动器的概念中，我们将每一个初始化代码抽象成了一个Task，然后，对它们进行了一个排序，根据它们之间的依赖关系排了一个有向无环图，接着，使用一个异步队列进行执行，并且这个异步队列它和CPU的核心数是强烈相关的，它能够最大程度地保证我们的主线程和别的线程都能够执行我们的任务，也就是大家几乎都可以同时完成。


## 3、启动优化有哪些容易忽略的注意点？

- **1、cpu time与wall time**
- **2、注意延迟初始化的优化**
- **3、介绍下黑科技**


首先，在CPU Profiler和Systrace中有两个很重要的指标，即cpu time与wall time，我们必须清楚cpu time与wall time之间的区别，wall time指的是代码执行的时间，而cpu time指的是代码消耗CPU的时间，锁冲突会造成两者时间差距过大。我们需要以cpu time来作为我们优化的一个方向。

其次，我们不仅只追求启动速度上的一个提升，也需要注意延迟初始化的一个优化，对于延迟初始化，通常的做法是在界面显示之后才去进行加载，但是如果此时界面需要进行滑动等与用户交互的一系列操作，就会有很严重的卡顿现象，因此我们使用了idealHandler来实现cpu空闲时间来执行耗时任务，这极大地提升了用户的体验，避免了因启动耗时任务而导致的页面卡顿现象。

最后，对于启动优化，还有一些黑科技，首先，就是我们采用了类预先加载的方式，我们在MultiDex.install方法之后起了一个线程，然后用Class.forName的方式来预先触发类的加载，然后当我们这个类真正被使用的时候，就不用再进行类加载的过程了。同时，我们再看Systrace图的时候，有一部分手机其实并没有给我们应用去跑满cpu，比如说它有8核，但是却只给了我们4核等这些情况，然后，有些应用对此做了一些黑科技，它会将cpu的核心数以及cpu的频率在启动的时候去进行一个暴力的提升。


## 4、版本迭代导致的启动变慢有好的解决方式吗？

- **启动器**
- **结合CI**
- **监控完善**


这种问题其实我们之前也遇到过，这的确非常难以解决。但是，我们后面对此进行了反复的思考与尝试，终于找到了一个比较好的解决方式。

首先，我们使用了启动器去管理每一个初始化任务，并且启动器中每一个任务的执行都是被其自动进行分配的，也就是说这些自动分配的task我们会尽量保证它会平均分配在我们每一个线程当中的，这和我们普通的异步是不一样的，它可以很好地缓解我们应用的启动变慢。

其次，我们还结合了CI，比如说，我们现在限制了一些类，如Application，如果有人修改了它，我们不会让这部分代码合并到主干分支或者是修改之后会有一些内部的工具如邮件的形式发送到我，然后，我就会和他确认他加的这些代码到底是耗时多少，能否异步初始化，不能异步的话就考虑延迟初始化，如果初始化时间太长，则可以考虑是否能进行懒加载，等用到的时候再去使用等等。

然后，我们会将问题尽可能地暴露在上线之前。同时，我们真正已经到了线上的一个环境下时，我们进行了监控的一个完善，我们不仅是监控了App的整个的启动时间，同时呢，我们也将每一个生命周期都进行了一个监控。比如说Application的onCreate与onAttachBaseContext方法的耗时，以及这两个生命周期之间间隔的时间，我们都进行了一个监控，如果说下一次我们发现了这个启动速度变慢了，我们就可以去查找到底是哪一个环节变慢了，我们会和以前的版本进行对比，对比完成之后呢，我们就可以来找这一段新加的代码。


# 七、总结

## 1、优化总方针

- **异步、延迟、懒加载**
- **技术、业务相结合**

## 2、注意事项

### 1、cpu time 和 wall time

- **wall time（代码执行时间）与cpu time（代码消耗CPU时间），锁冲突会造成这两者时间差距过大**。
- **cpu time才是优化方向，应尽力按照systrace的cpu time和wall time跑满cpu**。


### 2、监控的完善

- 线上监控多阶段时间（App、Activity、生命周期间隔时间）。
- 处理聚合看趋势。
- 收敛启动代码修改权限。
- 结合CI修改启动代码需要Review通知。


至此，探索Android启动速度优化的旅途也应该告一段落了，如果你耐心读到最后的话，会发现要想极致地提升App的性能，需要有一定的技术广度，如我们**引入了始于后端的AOP编程来实现无侵入式的函数插桩**，也需要有一定的深度，从前面的探索之旅来看，**我们先后涉及了Framework层、Native层、Dalvik虚拟机、甚至是Linux IO和文件系统相关的原理**。因此，我想说，**Android开发并不简单，即使是App层面的性能优化这一知识体系，也是需要我们不断地加深自身知识的深度和广度**。


    ps：在文章的黑科技部分涉及到了许多基础架构研发领域的知识，这部分无法理解的同学不要灰心，先了解即可，
        笔者之后的文章都会一一详细讲解。


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78bbd572ce5e4318a5162ae8f9a588c1~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、[Android开发高手课之启动优化](https://time.geekbang.org/column/article/73651)

2、[支付宝客户端架构解析：Android
客户端启动速度优化之「垃圾回收」](https://yq.aliyun.com/articles/672750)

3、[支付宝 App 构建优化解析：通过安装包重排布优化 Android 端启动性能](https://yq.aliyun.com/articles/680526)

4、[Facebook Redex字节码优化工具](https://github.com/facebook/redex)

5、[微信Android热补丁实践演进之路](https://blog.csdn.net/tencent_bugly/article/details/51821722)

6、[安卓App热补丁动态修复技术介绍](https://zhuanlan.zhihu.com/p/20308548)

7、[Dalvik Optimization and Verification With dexopt](https://blog.csdn.net/fishmai/article/details/52398485)

8、[微信在Github开源了Hardcoder，对Android开发者有什么影响？](https://github.com/Tencent/Hardcoder)

9、[历时三年研发，OPPO 的 Hyper Boost 引擎如何对系统、游戏和应用实现加速？](http://mobile.yesky.com/433/311567433.shtml)

10、[抱歉，Xposed真的可以为所欲为](https://blog.csdn.net/coder_pig/article/details/80031285)

11、[墙上时钟时间 ，用户cpu时间 ，系统cpu时间的理解](https://blog.csdn.net/balian8/article/details/53319580)  

12、《Android应用性能优化最佳实践》

13、[必知必会 | Android 性能优化的方面方面都在这儿](https://mp.weixin.qq.com/s/QVOYF2nfoWMCbM5YsxQgRQ?)

14、[极客时间之Top团队大牛带你玩转Android性能分析与优化](https://coding.imooc.com/class/308.html)

15、[启动器源码](https://github.com/zeshaoaaa/LaunchStarter)

16、[MultiDex优化源码](https://github.com/lanshifu/MultiDexTest)

17、[使用gradle自动化增加Trace Tag](https://github.com/AndroidAdvanceWithGeektime/Chapter07)


## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f179e593fc004ae9bce39875b29d57e3~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>


###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。
