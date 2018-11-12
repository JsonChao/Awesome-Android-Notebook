    <!-- GFM-TOC -->
* [一、组件化基础](#一组件化基础)
* [二、组件化编程](#二组件化编程)



## 一、 组件化基础

### 认识组件化

**多module划分为业务和基础功能**

- **组件**：指的是单一的功能组件，如视频组件（VideoSDK）、支付组件（PaySDK）、路由组件（Router）等，每个组件都能单独抽出来制作成SDK。
- **模块**: 指的是独立的业务模块，如直播模块（LiveModule）、首页模块（HomeModule）、即时通信模块（IMModule）等。模块相对于组件来说粒度更大，模块可能包含多种不同的组件。

**组件化开发的好处：**

- 避免重复造轮子，可以节省开发和维护的成本。
- 可以通过组件和模块为业务基准合理地安排人力，提高开发效率。
- 不同的项目可以共用一个组件或模块，确保整体技术方案的统一性。
- 为未来插件化共用同一套底层模型做准备。

**模块化开发的好处：**

- 业务模块的解耦，业务移植更加简单。
- 多团队根据业务内容进行并行开发和测试。
- 单个业务可以单独编译打包，加快编译速度。
- 多个App共用模块，降低了研发和维护成本。

**两者的缺点：**

- 旧项目重新适配组件化的开发需要相应的人力及时间成本。

**两者的本质思想：**

- 代码重用和业务解耦。

**区别：**

- 模块化是业务导向，组件化是功能导向。

引申：

项目方法数超过65535个时的解决方案：

- MultiDex分包。
- 插件化。

### 依赖

AS独有的三种依赖方式：

- **Jar dependency：** 通过Gradle配置引入lib文件夹中的所有.jar后缀的文件，还能引用.aar后缀的文件。
- **Base module：**
对应的是module dependency，实质上是将其打包成aar文件，方便其他库进行依赖。
- **Library dependency：**
第三方依赖通过其完成仓库索引依赖，仓库包含网络仓库和本地库。


    dependencies {
        compile fileTree（include：['*.jar'], dir: 'libs')
        compile project(':base')
        annotationProcessor 'com.alibaba:arouter-compiler:1.1.1'
    }

一般情况下，AS定义使用dependencies包含全部资源引入。

**注意：**

- 读入自身目录使用的是fileTree。
- 读入其他资源module使用的是”project“字段，而”：base“中冒号的意思是文件目录内与自己相同层级的其他module。


### 聚合和解耦

- AS正是以依赖的方式给每个module之间提供了沟通和交流的渠道，从而形成聚合。
- 聚合和解耦是项目架构的基础。
- 组件化架构就是在文件层级上有效地控制沟通和个体独立性的做法。


### 重新认识AndroidManifest

问题：每个module都有一份配合的AndroidManifest文件来记载其信息，最终生成一个App的时候，其只有一份AndroidManifest来指导App应该如何配置，那么如何记录这么多个module独立的配置信息呢？

**答案：将多个AndroidManifest合成一个**

合成的生成地址目录为app/build/intermediates/manifest/full/debug/AndroidManifest.xml, intermediates文件夹包含的是App生成过程中产生的“中间文件”。

**AndroidManifest属性变更**

**1.注册Activity**

    <application
        android:name="material.com.top.app.GankApplication"
        ...
        
name需要具体包名+属性名，这是因为AndroidManifest会引用多个module中的文件，需要知道具体路径，不然在编译期打包时会找不到每个文件的具体位置。

**2.注册Application**

如果功能module中有两个自定义Application，在解决冲突后，Application最终会载入后编译的module的Application。

**3.权限声明**

- 如果在一个功能module中声明所需要的权限，那么在主module中就会看到相应的权限。
- 如果在其他module中都声明相同的权限，最终的AndroidManifest会合并这个重复声明的权限，所以相同的权限只会声明一次。
- 如果考虑最终权限有可能被遗漏的问题，可以将全部的权限都在Base module中声明，这样全部权限都是有的。

**4.shareUid**

通过声明Shared User id，拥有同一个User id的多个App可以配置成运行在同一个进程中，所以默认可以互相访问任意数据。

问题：如果只是在功能module中声明shareUid，那么最终的AndroidManifest会如何呢？

**答案：只有在主module（Application module）中声明sharedUserId，才会最终打包到full AndroidManifest中。**

**注意：** 

- 每个module打包aar时都会将versionCode和versionName补全。


### 你所不知道的Application

**Applicaiton的基础和作用**

Application是整个App的一个单例对象，并且其生命周期是最长的，相当于整个App的生命周期。

Application中比较重要的方法：

- onTerminate——当终止应用程序对象时调用，不保证一定被调用，当程序被内核终止以便为其他应用程序释放资源时将不会提醒，并且不调用应用程序对象的onTerminate方法而直接终止进程。
- onLowMemory——当后台程序已经终止且资源还匮乏时会调用这个方法。好的应用程序会在此释放资源。

Applicaiton提供的最好用的方法：

- registerActivityLifecycleCallbacks()和unregisterActivityLifecycleCallbacks()。

作用：用于注册或注销对**App内所有Activity的生命周期监听。**

**组件化Application**

如果Library项目中也定义了与主项目相同的属性（例如默认生成的android:icon
和android:theme)，则此时会合并失败。

解决方式：使用tools：replace=“android:name"来声明Application是可被替换的。

在full文件夹中的AndroidManifest查看最终编入的是哪个Application。

### 小结

授人以鱼不如授人以渔，理解技能的基础原理，你会有一种一眼能看穿表层，直达里层的运转轨迹的感觉。当深入技能的高深层次的时候，更加需要加深对基础知识的理解。


## 二、组件化编程

### 本地广播

#### 对比全局广播和本地广播：

- 本地广播比全局广播要快，而且最接近于Android原生（出于Android.support兼容库）。
- 经过了多个support版本的迭代，稳定性和兼容性最优。
- 通信安全性、保密性和通信效率远高于全局广播。


#### 源码分析

本地广播使用了观察者的设计模式

先分析register注册代码：

    public void registerReciver(BroadccastReceiver receiver, IntentFilter filter) {
        synchronized (mReceivers) {
            //构造广播信息体
            ReceiverRecord entry = new ReceiverRecord(filter, receiver);
            ArrayList<IntentFilter> filters = mReceivers.get(receiver);
            if (filters == null) {
                filters = new ArrayList<IntentFilter>(1);
                //添加接收者接收信息
                mReceivers.put(receiver, filters);
            }
            filters.add(filter);
            for (int i = 0; i < filter.countActions(); i++) {
                String action = filter.getAction(i);
                ArrayList<ReceiverRecord> entries = mActions.get(action);
                if (entries == null) {
                    entries = new ArrayList<ReceiversRecord>(1);
                    //添加时间关联
                    mActions.put(action, entries);
                }
                entries.add(entry);
            }
        }
    }

构建一个ReceiverRecord广播信息实体，然后添加到广播数组列表Actions中。

放送广播实际操作源码如下：

    private void executePendingBroadcasts() {
        while (true) {
            BroadcastRecord[] brs = null;
            synchronized (mReceivers) {
                final int N = mPendingBroadcasts.size();
                if (N <= 0) {
                    return;
                }
                brs = new BroadcastRecord[N];
                mPendingBroadcasts.toArray(brs);
                mPendingBroadcasts.clear();
            }
            for (int i = 0; i < brs.length; i++) {
                //取得需要监听的广播的信息体
                BroadcastRecord br = brs[i];
                for (int j = 0; j < br.receivers.size(); j++) {
                    //取得接收对象调用onReceive方法
                    br.receivers.get(j).receiver.onReceive(mAppContext, br.intent);
                }
            }
        }
    }
    
- 调用sendBroadcast，传输广播Intent。
- 利用Intent中的Action索引广播数组列表，索引出广播实体。
- 利用Handler回调到主线程，调用executePendingBroadcasts来运行广播。
- 调用注册的BroadReciver的onReceive方法来运行广播出发内容。


### 组件间通信机制

#### 如何解决组件化的层级障碍？

如下图所示，通过使用Base module来跨越组件化层级，并且它也是模块间信息交流的基础。

![image](https://camo.githubusercontent.com/4a7d41d40aa16fadd0a96dfb3e3b1402e19c20b7/687474703a2f2f6f64677739633933692e626b742e636c6f7564646e2e636f6d2f2f467658506c615167416443676b56324c73475148786e743457423151)

#### 事件总线

##### 什么是事件总线？

如下图所示，事件总线机制通过记录对象、使用观察者模式来通知对象各种事件。

![image](https://upload-images.jianshu.io/upload_images/2276275-c20610cdd44f4c5f.png?imageMogr2/auto-orient/)

##### 为什么使用事件总线替代广播？

- 广播：耗时、容易被捕获（不安全）。
- 事件总线：更节省资源、更高效，能将信息传递给原生以外的各种对象。

##### 事件总线 EventBus

- 优点：开销小，代码更优雅、简洁，解耦发送者和接收者，可动态设置事件处理线程和优先级。
- 缺点：每个事件必须自定义一个事件类，增加了维护成本。

EventBus的观察者模式和一般的观察者模式不同，它使用了EventBus来作为中介者，抽离了许多职责，它的原理图如下：

![image](https://asce1885.gitbooks.io/android-rd-senior-advanced/content/EventBus.png)

##### 为何要解注册？

因为register是强引用，它会让对象无法得到内存回收，导致内存泄露。

##### EventBus3.0和EventBus2.x的区别？

- EventBus2.x使用的是运行时注解，它采用了反射的方式对整个注册的类的所有方法进行扫描来完成注册，因而会对性能有一定影响。
- EventBus3使用的是编译时注解，Java文件会编译成.class文件，在对class文件进行打包等一系列处理。在编译成.class文件时，EventBus会使用EventBusAnnotationProcessor注解处理器读取@Subscribe()注解并解析、处理其中的信息，然后生成Java类来保存所有订阅者的订阅信息。这样就创建出了对文件或类的索引关系，并将其编入到apk中。
- EventBus3.0使用了对象池缓存减少了创建对象的开销。

##### EventBus和RxBus的区别？

- RxJava的Observable有onError、onComplete等状态回调。
- Rxjava使用组合的而非嵌套的方式，避免了回调地狱。
- Rxjava的线程调度设计的更加优秀，更简单易用。
- Rxjava可使用多种操作符来进行链式调用来实现复杂的逻辑。
- Rxjava的信息效率高于EventBus2.x，低于EventBus3。

##### 建议

项目中使用了RxJava，则使用RxBus，否则使用EventBus3。

#### 组件化事件总线的考量

组件化要求功能模块独立，为了尽量少影响到App Module和Base Module，建议将事件总线功能单独做一个Module，让Base Module依赖即可。

### 组件间跳转

#### 隐式跳转

当使用包名+类名的方式来进行隐式跳转时，setClassName和setComponentName的第一个参数为APP包名。这是因为所有的AndoridManifest最终会合成一份AndroidManifest，这个时候模块已经消失了，Activity跳转需要索引包名。

#### ARouter路由跳转

##### 什么是路由？

是指路由器从一个接口收到数据包，根据数据路由包的目的地址进行定向转发到另一个接口的过程。Router（路由器）就是连接各个模块中页面跳转的中转站。

##### 原生跳转和路由跳转的对比

对比图如下所示。

![image](https://upload-images.jianshu.io/upload_images/11195468-82fcac93084628d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

##### ARouter组件化配置步骤

- 1.首先在Base module中添加一些配置：


    implementaion 'com.alibaba:arouter-api:1.1.0'
    annotationProcessor 'com.alibaba:arouter-compiler:1.1.1'
    
- 2.annotationProcessor会使用javaCompileOptions这个配置来获取当前module的名字。在各个模块中的build.gradle的defaultConfig属性中加入：


    javaCompileOptions {
        annotaionProcessorOptions {
            arguments = [ moduleName : project.getName() ]
        }
    }
    
- 3.每个模块的dependencies属性需要ARouter apt的引用，不然无法在apt中生成索引文件，无法跳转成功。


    dependencies {
        annotationProcessor 'com.alibaba:arouter-compiler:1.1.1'
    }
    
- 4.最后，在Application中初始化。


    @Override
    public void onCreate() {
        super.onCreate();
        if (Build.Config.DEBUG) {
            ARouter.openLog();
            ARouter.openDebub();
        }
        ARouter.init(this);
    }
    
##### ARouter使用步骤

- 1.给目的Activity添加注解Route，path是跳转的路径：

    
    // 注意：每个module的group名(即gank_web)不应该相同
    @Route(path = "/gank_web/1")
    public class WebActivity extends BaseActivity
    
- 2.使用ARouter跳转：

    
    ARouter.getInstance().build("/gank_web/1")
            .withString("url", url)
            .withString("title", desc)
            .navigation();
            
- 3.目的Activity读取传递的intent的方式就可以获取参数了：


    baseIntent = getIntent();
    String title = baseIntent.getStringExtra("title");
    String url = baseIntent.getStringExtra("url");
    
#### ARouter路由原理

1.ARouter拥有自身的编译时注解框架，编译期会根据注解生成三个文件，用于每个模块中页面的路由索引。其生成的三个文件继承于ARouter的接口，如下：

- Group(IRouteGroup)
- Providers(IProviderGroup)
- Root(IRouteRoot)

2.在Application加载的时候，ARouter会使用初始化调用init方法，之后再LogisticsCenter中通过加载编译时注解时创建的Group、Providers、Root三个类型的文件，使用Warehoure将文件保存到三个不同的HashMap对象中，Warehouse就相当于路由表，其保存着全部的模块跳转关系，如下所示：

![image](https://raw.githubusercontent.com/JsonChao/Awesome-Android-Notebook/master/screenshots/ARouter%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B%20.png)

3.ARouter路由跳转实际上还是调用了startActivity的跳转，使用了原生的Framework机制，只是通过apt注解的形式制造出跳转规则，并人为地拦截跳转和设置跳转条件，ARouter跳转流程图如下所示：

![image](https://raw.githubusercontent.com/JsonChao/Awesome-Android-Notebook/master/screenshots/ARouter%20%E8%B7%B3%E8%BD%AC%E8%BF%87%E7%A8%8B.png)

#### 组件化