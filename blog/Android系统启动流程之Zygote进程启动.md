---

		title:  Android系统启动流程之Zygote进程启动
		date: 2019/2/24 18:25:00   
		tags: 
		- Android核心源码分析
		categories: Android核心源码分析
		thumbnail: https://cdn.stemcellthailand.org/wp-content/uploads/2015/05/07stem-cells.jpg
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

在上一篇中，我们已经分析过Android系统启动流程中的init进程启动部分。在这一篇中，我们将继续分享Android系统启动流程中的Zygote进程启动部分。我们先看看Gityuan博客中的一幅系统启动架构图来对Android系统的启动流程有一个宏观的把控。

![image](http://gityuan.com/images/android-process/android-boot.jpg)

从图中可得知Android系统中各个进程的先后顺序为：

init进程 –-> Zygote进程 –> SystemServer进程 –>应用进程

其中Zygote进程由init进程启动，这篇文章将从以下三部分来对Zygote进行分析：

- 1、Zygote是什么？
- 2、Zygote启动脚本
- 3、Zygote进程启动流程


### 一、Zygote是什么？

Zygote是在init进程启动时创建的，它又称为孵化器，它可以通过fork（复制进程）的形式来创建应用程序进程和SystemServer进程。并且，Zygote进程在启动的时候回创建DVM或者ART，因此通过fork而创建的应用程序进程和SystemServer进程可以在内部获取一个DVM或者ART的实例副本。


### 二、Zygote启动脚本

init.rc文件中采用了如下所示的Import类型语句来引入Zygote启动脚本：

    import /init.${ro.zygote}.rc
    

这里根据属性ro.zygote的内容来引入不同的Zygote启动脚本。从Android 5.0开始，Android开始支持64位程序，Zygote有了32/64位之别，ro.zygote属性的取值有4种：

- init.zygote32.rc
- init.zygote32_64.rc
- init.zygote64.rc
- init.zygote64_32.rc

注意：上面的Zygote的启动脚本都存放在system/core/rootdir目录中。

下面我们来一一分析一下上述的Zygote启动脚本。

#### 1、init.zygote32.rc

仅支持32位程序，脚本源码如下所示：

    service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
        class main
        priority -20
        user root
        group root readproc reserved_disk
        socket zygote stream 660 root system
        onrestart write /sys/android_power/request_state wake
        onrestart write /sys/power/state on
        onrestart restart audioserver
        onrestart restart cameraserver
        onrestart restart media
        onrestart restart netd
        onrestart restart wificond
        writepid /dev/cpuset/foreground/tasks


可以看到，它就是Android初始化语言的Service类型语句，格式如下：

    service <name> <pathname> [ <argument> ]*   //<service的名字><执行程序路径><传递参数>  
        <option>       //option是service的修饰词，影响什么时候、如何启动services  
        <option>  
        ...

由此，我们可以知道Zygote进程的名字为zygote，执行程序路径为/system/bin/app_process，类名为main，上述类似“onrestart restart ...”格式的语句表示如果audioserver、cameraserver、media等进程终止了，就会进行重启。


#### 2、init.zygote32_64.rc

同时支持32、64位程序，脚本源码如下所示：

    service zygote /system/bin/app_process32 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
        class main
        priority -20
        user root
        group root readproc reserved_disk
        socket zygote stream 660 root system
        onrestart write /sys/android_power/request_state     wake
        onrestart write /sys/power/state on
        onrestart restart audioserver
        onrestart restart cameraserver
        onrestart restart media
        onrestart restart netd
        onrestart restart wificond
        writepid /dev/cpuset/foreground/tasks

    service zygote_secondary /system/bin/app_process64 -Xzygote /system/bin --zygote     --socket-name=zygote_secondary
        class main
        priority -20
        user root
        group root readproc reserved_disk
        socket zygote_secondary stream 660 root system
        onrestart restart zygote
        writepid /dev/cpuset/foreground/tasks


可以看出，这里使用了两个Service类型语句启动了两个Zygote进程，一个是名字为zygote，执行程序为app_process32的主模式Zygote进程；另一个是名字为zygote_secondary，执行程序为app_process64的辅模式Zygote进程。另外的init.zygote64.rc和init.zygote64_32.rc与上面的Zygote脚本都是类似的，这里不再多说了。


### 三、Zygote进程启动流程

在init启动Zygote时主要是调用app_main.cpp的main函数中的AppRuntime.start()方法来启动Zygote进程的，我们先看到app_main.cpp的main函数：

    int main(int argc, char* const argv[])
    {
        ...
        while (i < argc) {
            const char* arg = argv[i++];
            // 1
            if (strcmp(arg, "--zygote") == 0) {
                zygote = true;
                niceName = ZYGOTE_NICE_NAME;
            } else if (strcmp(arg, "--start-system-server") == 0) {
                // 2
                startSystemServer = true;
            } else if (strcmp(arg, "--application") == 0) {
                // 3
                application = true;
            } else if (strncmp(arg, "--nice-name=", 12) == 0) {
                niceName.setTo(arg + 12);
            } else if (strncmp(arg, "--", 2) != 0) {
                className.setTo(arg);
                break;
            } else {
                --i;
                break;
            }
        }

        ...

        // 4
        if (zygote) {
            runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
        } else if (className) {
            runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
        } else {
            fprintf(stderr, "Error: no class name or --zygote supplied.\n");
            app_usage();
            LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        }
    }


由前可知，Zygote进程都是通过fork自身来创建子进程的，这样Zygote进程和由它fork出来的子进程都会进入app_main.cpp的main函数中，所以在mian函数中，首先会判断当前运行在哪个进程，在注释1处，会判断参数arg中释放包含了"--zygote"，如果包含了，则说明main函数是运行在Zygote进程中的并会将zygote标记置为true。在注释2处会判断参数arg中是否包含了"--start-system-server"，如果包含了则表示当前是处在SystemServer进程中并将startSystemServer设置为true。同理在注释3处会判断参数arg是否包含"--application"，如果包含了说明当前处在应用程序进程中并将application标记置为true。最后在注释4处，当zygote标志是true的时候，也就是当前正处在Zygote进程中时，则使用AppRuntime.start()函数启动Zygote进程。

我们接着看看AndroidRuntime的start函数：

    void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
    {
        ...
        
        /* start the virtual machine */
        JniInvocation jni_invocation;
        jni_invocation.Init(NULL);
        JNIEnv* env;
        // 1
        if (startVm(&mJavaVM, &env, zygote) != 0) {
            return;
        }
        onVmCreated(env);

        /*
         * 2、Register android functions.
         */
        if (startReg(env) < 0) {
            ALOGE("Unable to register all android natives\n");
            return;
        }

        ...
        // 3
        classNameStr = env->NewStringUTF(className);
        assert(classNameStr != NULL);
        env->SetObjectArrayElement(strArray, 0, classNameStr);

        for (size_t i = 0; i < options.size(); ++i) {
            jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
            assert(optionsStr != NULL);
            env->SetObjectArrayElement(strArray, i + 1, optionsStr);
        }

        /*
         * Start VM.  This thread becomes the main thread of the VM, and will
         * not return until the VM exits.
         */
        // 4
        char* slashClassName = toSlashClassName(className != NULL ? className : "");
        jclass startClass = env->FindClass(slashClassName);
        if (startClass == NULL) {
            ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
            /* keep going */
        } else {
            // 6
            jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
                "([Ljava/lang/String;)V");
            if (startMeth == NULL) {
                ALOGE("JavaVM unable to find main() in '%s'\n", className);
                /* keep going */
            } else {
                // 6
                env->CallStaticVoidMethod(startClass, startMeth, strArray);

    #if 0
                if (env->ExceptionCheck())
                    threadExitUncaughtException(env);
    #endif
            }
        }
        free(slashClassName);

        ...
    }
    

首先，在AndroidRuntime的start函数中，会现在注释1处使用startVm函数来启动弄Java虚拟机，然后在注释2处使用startReg函数为Java虚拟机注册JNI方法。在注释3处的classNameStr是传入的参数，值为com.android.internall.os.ZygoteInit。然后在注释4处使用toSlashClassName函数将className的"."替换为"/"，替换后的值为com/android/internal/os/ZygoteInit。接着根据这个值找到ZygoteInit并在注释5处找到ZygoteInit的main函数，最后在注释6处使用JNI调用ZygoteInit的main函数，之所以这里要使用JNI，是因为ZygoteInit是java代码。最终，Zygote就从Native层进入了Java FrameWork层。在此之前，并没有任何代码进入Java FrameWork层面，因此可以认为，Zygote开创了java FrameWork层。

接着，我们看看Zygoteinit.java中的main方法：

     public static void main(String argv[]) {
            
        ...

        try {
            ...
            
            // 1
            zygoteServer.registerServerSocketFromEnv(socketName);
            // In some configurations, we avoid preloading resources and classes eagerly.
            // In such cases, we will preload things prior to our first fork.
            if (!enableLazyPreload) {
                bootTimingsTraceLog.traceBegin("ZygotePreload");
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                    SystemClock.uptimeMillis());
                    
                // 2
                preload(bootTimingsTraceLog);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                    SystemClock.uptimeMillis());
                bootTimingsTraceLog.traceEnd(); // ZygotePreload
            } else {
                Zygote.resetNicePriority();
            }

            ...
            
            if (startSystemServer) {
                // 3
                Runnable r = forkSystemServer(abiList, socketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }

            Log.i(TAG, "Accepting command socket connections");

            // The select loop returns early in the child process after a fork and
            // loops forever in the zygote.
            // 4
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            throw ex;
        } finally {
            zygoteServer.closeServerSocket();
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }
    

首先，在注释1处调用了ZygoteServer的registerServerSocketFromEnv方法创建了一个名为"zygote"的Server端的Socket，它用来等待ActivityManagerService请求Zygote来创建新的应用程序进程。我首先分析下registerServerSocketFromEnv方法的处理逻辑，源码如下所示：

    private static final String ANDROID_SOCKET_PREFIX = "ANDROID_SOCKET_";

    void registerServerSocketFromEnv(String socketName) {
        if (mServerSocket == null) {
            int fileDesc;
            // 1
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                // 2
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                // 3
                mServerSocket = new LocalServerSocket(fd);
                mCloseSocketFd = true;
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }


首先，会在注释1处将Socket的名字拼接为“ANDROID_SOCKET_zygote“，在注释2处调用System.getenv()方法得到该Socket对应的环境变量中的值，然后将这个Socket环境变量值解析为int类型的文件描述符参数。接着，在注释4处，使用上面得到的文件描述符参数得到一个文件描述符，并由此新建一个服务端Socket。当Zygote进程将SystemServer进程启动红藕，就会在这个服务端Socket上等待AMS请求Zygote进程去创建新的应用程序进程。

接着，我们回到ZygoteInit的main方法，在注释2处会预加载类和资源。然后在注释3处，使用了forkSystemServer()方法去创建SystemServer进程。forkSystemServer()方法核心代码如下所示：

     private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
        
        // 一系统创建SystemServer进程所需参数的准备工作
        
        try {
            ...
            
            /* Request to fork the system server process */
            // 3.1
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
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            zygoteServer.closeServerSocket();
            
            // 3.2
            return handleSystemServerProcess(parsedArgs);
        }

        return null;
    }


可以看到，forkSystemServer()方法中，注释3.1调用了Zygote的forkSystemServer()方法去创建SystemServer进程，其内部会执行nativeForkSystemServer这个Native方法，它最终会使用fork函数在当前进程创建一个SystemServer进程。如果pid等于0，即当前是处于新创建的子进程ServerServer进程中，则在注释3.2处使用handleSystemServerProcess()方法处理SystemServer进程的一些处理工作。

我们再回到Zygoteinit.java中main方法中的注释4处，这里调用了ZygoteServer的runSelectLoop方法来等等ActivityManagerService来请求创建新的应用程序进程，runSelectLoop()方法如下所示：

     Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        // 1
        fds.add(mServerSocket.getFileDescriptor());
        peers.add(null);

        // 2、无限循环等待AMS请求创建应用程序进程
        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            // 3
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }

                // 4
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    try {
                        ZygoteConnection connection = peers.get(i);
                        
                        // 5
                        final Runnable command = connection.processOneCommand(this);

                        if (mIsForkChild) {
                            // We're in the child. We should always have a command to run at this
                            // stage if processOneCommand hasn't called "exec".
                            if (command == null) {
                                throw new IllegalStateException("command == null");
                            }

                            return command;
                        } else {
                            // We're in the server - we should never have any commands to run.
                            if (command != null) {
                                throw new IllegalStateException("command != null");
                            }

                            // We don't know whether the remote side of the socket was closed or
                            // not until we attempt to read from it from processOneCommand. This shows up as
                            // a regular POLLIN event in our regular processing loop.
                            if (connection.isClosedByPeer()) {
                                connection.closeSocket();
                                peers.remove(i);
                                fds.remove(i);
                            }
                        }
                    } catch (Exception e) {
                        if (!mIsForkChild) {
                            // We're in the server so any exception here is one that has taken place
                            // pre-fork while processing commands or reading / writing from the
                            // control socket. Make a loud noise about any such exceptions so that
                            // we know exactly what failed and why.

                            Slog.e(TAG, "Exception executing zygote command: ", e);

                            // Make sure the socket is closed so that the other end knows immediately
                            // that something has gone wrong and doesn't time out waiting for a
                            // response.
                            ZygoteConnection conn = peers.remove(i);
                            conn.closeSocket();

                            fds.remove(i);
                        } else {
                            // We're in the child so any exception caught here has happened post
                            // fork and before we execute ActivityThread.main (or any other main()
                            // method). Log the details of the exception and bring down the process.
                            Log.e(TAG, "Caught post-fork exception in child process.", e);
                            throw e;
                        }
                    } finally {
                        // Reset the child flag, in the event that the child process is a child-
                        // zygote. The flag will not be consulted this loop pass after the Runnable
                        // is returned.
                        mIsForkChild = false;
                    }
                }
            }
        }
    }
    

首先，在注释1处，会调用服务端的mServerSocket的getFileDescriptor()函数来去获得自身的fd字段值并加入fds列表中。然后，在注释2处，无限循环用来等待AMS请求Zygote进程创建新的应用程序进程。在注释3处会遍历pollFds这个fd列表，如果i等于0，则说明服务端Socket与客户端连接上了，即当前Zygote进程与AMS进程建立了连接。接着，在注释4处调用acceptCommandPeer()方法得到ZygoteConnection对象，并将其加入peers列表中。如果i不等于0，则表明AMS想Zygote进程发送了一个创建应用程序进程的请求，最后会在注释5处执行ZygoteConnection.runOnce方法去创建一个新的应用程序进程。


### 四、总结

从以上的分析可以得知，Zygote进程启动中承担的主要职责如下：

- 1、创建AppRuntime，执行其start方法，启动Zygote进程。。
- 2、创建JVM并为JVM注册JNI方法。
- 3、使用JNI调用ZygoteInit的main函数进入Zygote的Java FrameWork层。
- 4、使用registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等等AMS的请求去创建新的应用进程。
- 5、启动SystemServer进程。

至此，Android系统启动流程之Zygote进程启动部分分析完毕，下一篇将会详细分析SystemServer进程启动相关的部分，敬请期待！



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