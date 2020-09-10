---

		title:  Android系统启动流程之SystemServer进程启动
		date: 2019/3/03 22:32:00   
		tags: 
		- Android核心源码分析
		categories: Android核心源码分析
		thumbnail: https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcT4VXnyYZyEtzXFyDDCSUxlnA6RaSnp3_Qq7oGZfHghM7Pc3B7d
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。


在上一篇中，笔者已经分析过Android系统启动流程中的init进程和Zygote进程启动部分。Android系统中各个进程的先后顺序为：

init进程 –-> Zygote进程 –> SystemServer进程 –>应用进程

其中Zygote进程由init进程启动，SystemServer进程和应用进程由Zygote进程启动。在这一篇中，我将继续分析Android系统启动流程中的SystemServer进程启动部分。

SystemServer进程主要是用于创建系统服务的，例如AMS、WMS、PMS。这篇文章将从以下两个部分来对SystemServer进行分析：

- Zygote处理SystemServer进程
- SystemServer进程解析

### 一、Zygote处理SystemServer进程

由前文可知，在ZygoteInit的forkSystemServer()方法中启动了SystemServer进程，如下所示：

    private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
            
        ...
        
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            boolean profileSystemServer = SystemProperties.getBoolean(
                    "dalvik.vm.profilesystemserver", false);
            if (profileSystemServer) {
                parsedArgs.runtimeFlags |= Zygote.PROFILE_SYSTEM_SERVER;
            }

            /* Request to fork the system server process */
            // 1
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.runtimeFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        // 2
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
            // 3
            zygoteServer.closeServerSocket();
            // 4
            return handleSystemServerProcess(parsedArgs);
        }

        return null;
    }


在注释1处，调用了Zygote的forkSystemServer()方法创建了SystemServer进程，并返回了当前进程的pid。在注释2处，如果pid==0则说明Zygote进程创建SystemServer进程成功，当前运行在SystemServer进程中。接着，在注释3处，由于SystemServer进程fork了Zygote进程的地址空间，所以会得到Zygote进程创建的Socket，这个Socket对于SystemServer进程是无用的，因此，在此处关闭了该Socket。最后，在注释4处，调用了handleSystemServerprocess()方法来启动SystemServer进程。handleSystemServerProcess()方法如下所示：


    /**
     * Finish remaining work for the newly forked system server process.
     */
    private static Runnable handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
    
        ...
        
        if (parsedArgs.invokeWith != null) {
        
            ...
        } else {
            ClassLoader cl = null;
            if (systemServerClasspath != null) {
                // 1
                cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);

                Thread.currentThread().setContextClassLoader(cl);
            }

            /*
             * Pass the remaining arguments to SystemServer.
             */
            // 2
            return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
        }
    }


在注释1处，使用了systemServerClassPath和targetSdkVersion创建了一个PathClassLoader。接着，在注释2处，执行了ZygoteInit的zygoteInit()方法，该方法如下所示：


    public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
        if (RuntimeInit.DEBUG) {
            Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
        RuntimeInit.redirectLogStreams();

        RuntimeInit.commonInit();
        // 1
        ZygoteInit.nativeZygoteInit();
        // 2
        return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
    }


在zygoteInit()方法中，首先在注释1处执行了nativeZygoteInit()方法，这里看到方法前缀为native可知是一个本地函数，因此，我们先了解它对应的JNI文件，在AndroidRuntime.cpp类中可以查看到nativeZygoteInit()方法对应的native函数，如下所示：


    /*
    * JNI registration.
    */
    int register_com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env)
    {
        const JNINativeMethod methods[] = {
            { "nativeZygoteInit", "()V",
                (void*) com_android_internal_os_ZygoteInit_nativeZygoteInit },
        };
        return jniRegisterNativeMethods(env, "com/android/internal/os/ZygoteInit",
            methods, NELEM(methods));
    }
    
    
这里使用了JNI动态注册的方式，将nativeZygoteInit()方法和native函数com_android_internal_os_ZygoteInit_nativeZygoteInit()建立了映射关系，我们看到这个native方法的代码：

    static AndroidRuntime* gCurRuntime = NULL;

    static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
    {
        gCurRuntime->onZygoteInit();
    }
    
    
可以看到，gCurRuntime是AndroidRuntime类型的指针，具体指向的是其子类AppRuntime，它在app_main.cpp中定义，代码如下所示：

    class AppRuntime : public AndroidRuntime
    {
        ...

        virtual void onZygoteInit()
        {
            // 1
            sp<ProcessState> proc = ProcessState::self();
            ALOGV("App process: starting thread pool.\n");
            // 2
            proc->startThreadPool();
        }
        
        ...
    }


在注释1处，创建了一个ProcessState实例， 在Android中ProcessState是客户端和服务端公共的部分，作为Binder通信的基础，ProcessState是一个singleton类，每个
进程只有一个对象，这个对象负责打开Binder驱动，建立线程池，让其进程里面的所有线程都能通过Binder通信。在注释2处，调用了ProcessState实例的startThreadPool()函数启动了一个Binder线程池，其实里面最终会调用到IPCThreadState实例的joinThreadPool()函数进程Binder线程池相关的处理。现在，我们再回到zygoteInit()方法的注释2处，这里调用了RuntimeInit的applicationInit()方法，代码如下所示：


    protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
        ClassLoader classLoader) {
    
        ...
        
        // Remaining arguments are passed to the start class's static main
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }
    

在applicationInit()方法中最后调用了findStaticMain()方法：


    protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Class<?> cl;

        try {
            // 1
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            // 2
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        // 3
        return new MethodAndArgsCaller(m, argv);
    }
    
    
首先，在注释1处，通过发射得到了SystemServer类。接着，在注释2处，找到了SystemServer中的main()方法。最后，在注释3处，会将main()方法传入MethodAndArgsCaller()方法中，这里的MethodAndArgsCaller()方法是一个Runnable实例，它最终会一直返回出去，直到在ZygoteInit的main()方法中被使用，如下所示：

    if (startSystemServer) {
        Runnable r = forkSystemServer(abiList, socketName, zygoteServer);

        // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
        // child (system_server) process.
        if (r != null) {
            r.run();
            return;
        }
    }


可以看到，最终直接调用了这个Runnable实例的run()方法，代码如下所示：


    /**
     * Helper class which holds a method and arguments and can call them. This is used as part of
     * a trampoline to get rid of the initial process setup stack frames.
     */
    static class MethodAndArgsCaller implements Runnable {
        /** method to call */
        private final Method mMethod;

        /** argument array */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                // 1
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    }
    

在注释1处，这个mMethod就是指的SystemServer的main()方法，这里动态调用了SystemServer的main()方法，最终，SystemServer进程就进入了SystemServer的main()方法中了。这里还有个遗留问题，为什么不直接在findStaticMain()方法中直接动态调用SystemServer的main()方法呢？原因就是这种递归返回后再执行入口方法的方式会让SystemServer的main()方法看起来像是SystemServer的入口方法，而且，这样也会清除之前所有SystemServer相关设置过程中需要的堆栈帧。


### 二、SystemServer进程解析

接下来我们看看SystemServer的main()方法：


    /**
    * The main entry point from zygote.
    */
    public static void main(String[] args) {
        new SystemServer().run();
    }


main()方法中调用了SystemServer的run()方法，如下所示：

    
    private void run() {
        try {
            ...
            
            // 1
            Looper.prepareMainLooper();
            ...
            
            // Initialize native services.
            // 2
            System.loadLibrary("android_servers");
            
            // Check whether we failed to shut down last time we tried.
            // This call may not return.
            performPendingShutdown();
            
             // Initialize the system context.
            createSystemContext();

            // Create the system service manager.
            // 3
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
            // Prepare the thread pool for init tasks that can be parallelized
            SystemServerInitThreadPool.get();
        } finally {
            traceEnd();  // InitBeforeStartServices
        }

        // Start services.
        try {
            traceBeginAndSlog("StartServices");
            // 4
            startBootstrapServices();
            // 5
            startCoreServices();
            // 6
            startOtherServices();
            SystemServerInitThreadPool.shutdown();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            traceEnd();
        }
        
        ...
        
        // Loop forever.
        // 7
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
    
        
在注释1处，创建了消息Looper。在注释2处，加载了动态库libandroid_servers.so。接着，在注释3处，创建了SystemServerManager，它的作用是对系统服务进行创建、启动和生命周期管理。在注释4处的startBootstarpServices()方法中使用SystemServiceManager启动了ActivityManagerService、PackageManagerService、PowerManagerService等引导服务。在注释5处的startCoreServices()方法中则启动了BatteryService、WebViewUpdateService、DropBoxManagerService、UsageStatsService4个核心服务。在注释6处的startOtherServices()方法中启动了WindowManagerService、InputManagerService、CameraService等其它服务。这些服务的父类都是SystemService。

可以看到，上面把系统服务分成了三种类型：引导服务、核心服务、其它服务。这些系统服务共有100多个，其中对于我们来说比较关键的有：

- 引导服务：ActivityManagerService，负责四大组件的启动、切换、调度。
- 引导服务：PackageManagerService，负责对APK进行安装、解析、删除、卸载等操作。
- 引导服务：PowerManagerService，负责计算系统中与Power相关的计算，然后决定系统该如何反应。
- 核心服务：BatteryService，管理电池相关的服务。
- 其它服务：WindowManagerService，窗口管理服务。
- 其它服务：InputManagerService，管理输入事件。

很多系统服务的启动逻辑都是类似的，这里我以启动ActivityManagerService服务来进行举例，代码如下所示：


    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
            
    
SystemServiceManager的startService()方法启动了ActivityManagerService，该启动方法如下所示：


    @SuppressWarnings("unchecked")
    public <T extends SystemService> T startService(Class<T> serviceClass) {
        try {
            final String name = serviceClass.getName();
            
            ...
            
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                // 1
                service = constructor.newInstance(mContext);
            } catch (InstantiationException ex) {
            
            ...
            
            // 2
            startService(service);
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }
    
    
在注释1处使用反射创建了ActivityManagerService实例，并在注释2处调用了另一个startService()重载方法，如下所示：


    public void startService(@NonNull final SystemService service) {
        // Register it.
        // 1
        mServices.add(service);
        // Start it.
        long time = SystemClock.elapsedRealtime();
        try {
            // 2
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + service.getClass().getName()
                    + ": onStart threw an exception", ex);
        }
        warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
    }
    
    
在注释1处，首先会将ActivityManagerService添加在mServices中，它是一个存储SystemService类型的ArrayList，这样就完成了ActivityManagerService的注册。在注释2处，调用了ActivityManagerService的onStart()方法完成了启动ActivityManagerService服务。

除了使用SystemServiceManager的startService()方法来启动系统服务外，也可以直接调用服务的main()方法来启动系统服务，如PackageManagerService：


    mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    
    
这里直接调用了PackageManagerService的main()方法：

    
    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();

        // 1
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        // 2
        ServiceManager.addService("package", m);
        // 3
        final PackageManagerNative pmn = m.new PackageManagerNative();
        ServiceManager.addService("package_native", pmn);
        return m;
    }


在注释1处，直接新建了一个PackageManagerService实例，并在注释2处将PackageManagerService注册到服务大管家ServiceManager中，ServiceManager用于管理系统中的各种Service，用于系统C/S架构中的Binder进程间通信，即如果Client端需要使用某个Servcie，首先应该到ServiceManager查询Service的相关信息，然后使用这些信息和该Service所在的Server进程建立通信通道，这样Client端就可以服务端进程的Service进行通信了。


### 三、总结

SystemService的启动流程分析至此已经完结，经过以上的分析可知，SystemService进程被创建后，主要的处理如下：

- 1、启动Binder线程池，这样就可以与其他进程进行Binder跨进程通信。
- 2、创建SystemServiceManager，它用来对系统服务进行创建、启动和生命周期管理。
- 3、启动各种系统服务：引导服务、核心服务、其他服务，共100多种。应用开发主要关注引导服务ActivityManagerService、PackageManagerService和其他服务WindowManagerService、InputManagerService即可。

下篇，将会给大家带来Android系统启动流程之Launcher进程启动的详细分析，希望大家多多支持~


##### 参考链接：
---
1、Android V9.0.0 源码

2、Android进阶解密第二章

3、[Android系统开篇](http://gityuan.com/android/)


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

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。。