---

		title:  Android系统启动流程之init进程启动
		date: 2019/2/18 01:00:00   
		tags: 
		- Android核心源码分析
		categories: Android核心源码分析
		thumbnail: https://ssl.tzoo-img.com/images/blog/legacyblog/cn/wp-content/uploads/2017/03/Hot-Balloon-2.jpg?width=412&spr=3
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

众所周知，Android核心源码这一块的知识一直是阻碍广大Android开发者成为高级甚至资深Android的拦路虎，因此，从这篇开始，笔者接下来将会陪大家深入分析Android中的核心源码，从而能够让我们真正地去理解Android源码背后的设计思想与艺术，真真切切地提升自己的内功。

Android系统启动流程共分为四篇，分别为：

- Android系统启动流程之init进程启动
- Android系统启动流程之Zygote进程启动
- Android系统启动流之SystemServer进程启动
- Android系统启动流程之Launcher进程启动


接下来，笔者将会为大家介绍Android系统启动流程的相关知识，但是由于篇幅太长，这里便打算分为四部分来进行讲解。这一篇，我将首先对其中最重要的init进程启动过程进行分析。这里，先给出一幅Gityuan博客中的一幅系统启动架构图来对Android系统的启动流程有一个宏观的把控。

![image](http://gityuan.com/images/android-arch/android-boot.jpg)


下面，我们正式开始进行分析~


## 一、启动电源以及系统启动

当电源按下时引导芯片代码会从预定义的地方（固化在ROM）开始执行，加载引导程序BootLoader到RAM，然后执行。


## 二、引导程序BootLoader

它是Android操作系统开始运行前的一个小程序，主要将操作系统OS拉起来并进行。


## 三、Linux内核启动

当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。此外，还启动了Kernel的swapper进程（pid = 0）和kthreadd进程（pid = 2）。下面分别介绍下它们：

- swapper进程：又称为idle进程，系统初始化过程Kernel由无到有开创的第一个进程, 用于初始化进程管理、内存管理，加载Binder Driver、Display、Camera Driver等相关工作。
- kthreadd进程：Linux系统的内核进程，是所有内核进程的鼻祖，会创建内核工作线程kworkder，软中断线程ksoftirqd，thermal等内核守护进程。

当内核完成系统设置时，它首先在系统文件中寻找init.rc文件，并启动init进程。


## 四、init进程启动

init进程主要用来初始化和启动属性服务，并且启动Zygote进程。


### 1、init进程是什么？

Linux系统的用户进程，是所有用户进程的鼻祖，进程号为1，它有许多重要的职责，比如创建Zygote孵化器和属性服务等。并且它是由多个源文件组成的，对应源码目录system/core/init中。


### 2、init进程启动核心代码流程分析

init进程的启动会首先从init进程的入口函数开始，init进程的入口函数main位于system/core/init/init.cpp中，代码如下所示：


    int main(int argc, char** argv) {
        
        ...
        // 如果是初始化第一阶段，则需要执行下面的步骤1
        if (is_first_stage) {
            ...
            
            // 清理umask
            umask(0);
            
            ...
            
            // 1、创建和挂载启动所需的文件目录
            mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
            mkdir("/dev/pts", 0755);
            mkdir("/dev/socket", 0755);
            mount("devpts", "/dev/pts", "devpts", 0, NULL);
            #define MAKE_STR(x) __STRING(x)
            mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
            ...
            
            // 初识化Kernel的Log，获取外界的Kernel日志
            InitKernelLogging(argv);
            
            ...
        }
        
        // 初识化Kernel的Log，获取外界的Kernel日志
        InitKernelLogging(argv);
        
        ...
        
        // 2、初始化属性相关资源
        property_init();
        
        ...
        
        // 创建epoll句柄
        epoll_fd = epoll_createl(EPOLL_CLOEXEC);
        ...
        
        // 3、设置子信号处理函数
        sigchld_handler_init();
        
        // 导入默认的环境变量
        property_load_boot_defaults();
        
        // 4、启动属性服务
        start_property_service();
        set_usb_controller();
        
        ...
        
        // 加载引导脚本
        LoadBootScripts(am, sm);
        
        ...   
        while (true) {
        
            ...
        
            if (!(waiting_for_prop || Service::is_exec_service_running())) {
                // 内部会偏离执行每个action中携带的command对应的执行函数
                am.ExecuteOneCommand();
                
            }
            if (!(waiting_for_prop || Service::is_exec_service_running())) {
                if (!shutting_down) {
                    // 重启死去的子进程
                    auto next_process_restart_time = RestartProcesses();
                    
                    ...
                }
                
                // If there's more work to do, wake up again immediately.
                if (am.HasMoreCommands()) epoll_timeout_ms = 0;
            }
            epoll_event ev;
            int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
            if (nr == -1) {
                PLOG(ERROR) << "epoll_wait failed";
            } else if (nr == 1) {
                ((void (*)()) ev.data.ptr)();
            }
        }
        
        return 0;
    }
    
    static void LoadBootScripts(ActionManager&action_manager, ServiceList& service_list) {
        Parser parser = CreateParser(action_manager, service_list);
    
        std::string bootscript = GetProperty("ro.boot.init_rc", "");
        // bootscript默认是空的
        if (bootscript.empty()) {
            // 5、解析init.rc配置文件
            parser.ParseConfig("/init.rc");
            if (!parser.ParseConfig("/system/etc/init")) {
                late_import_paths.emplace_back("/system/etc/init");
            }
            if (!parser.ParseConfig("/product/etc/init")) {
                late_import_paths.emplace_back("/product/etc/init");
            }
            if (!parser.ParseConfig("/odm/etc/init")) {
                late_import_paths.emplace_back("/odm/etc/init");
            }
            if (!parser.ParseConfig("/vendor/etc/init")) {
                late_import_paths.emplace_back("/vendor/etc/init");
            }
        } else {
            parser.ParseConfig(bootscript);
        }
    }


#### 1、创建和挂载启动所需的文件目录

其中挂载了tmpsf、devpts、proc、sysfs和selinuxfs共5种文件系统（它们均是系统运行时目录）：

    mount(...);
    mkdir(...);
    ...


#### 2、对属性服务进行初始化

    property_init();
    
    
##### 什么是属性服务？

Windows平台上有一个注册表管理器，注册表的内容采用键值对的形式来记录用户、软件等使用信息。如果系统或软件重启，还是能够根据这份注册表中的记录，进行相应的初识化工作。Android也提供了一个这样类型的机制，即属性服务。


##### 它具体是如何进行初始化的？

我们查看system/core/init/property_service.cpp源码中的property_init()函数：

    void property_init() {
        mkdir("/dev/__properties__", S_IRWXU | S_IXGRP S_IXOTH);
        CreateSerializedPropertyInfo();
        // 关注点
        if (__system_property_area_init()) {
            LOG(FATAL) << "Failed to initialize property area";
        }
        if (!property_info_area.LoadDefaultPath()) {
            LOG(FATAL) << "Failed to load serialized property info file";
        }
    }
    

init进程启动时会启动属性服务，并为其分配内存，用来存储这些属性，如果需要就可以直接读取，具体在代码里就是执行了property_init()函数中的__system_property_area_init()函数去初始化属性内存区域。


#### 3、设置子进程信号处理函数，如果子进程（zygote进程）异常退出，init进程会调用该函数中设定的信号处理函数来进行处理

    sigchld_handler_init();
    
    
##### sigchld_handler_init()的作用：

防止init进程的子进程成为僵尸进程，为了防止僵尸进程的出现，系统会在子进程暂停和终止的时候发出SIGCJHLD信号，该函数就是用来接收SIGCHLD信号的，注意它仅处理进程终止的SIGCHLD信号。


##### 僵尸进程是什么？

在UNIX/Linux中，父进程使用fork创建子进程，子进程终止后，如果父进程不知道子进程已经终止的话，这时子进程虽然已经退出，但是在系统进程表中还为它保留了一些信息（如进程号、运行时间、退出状态等），这个子进程就是所谓的僵尸进程。其中系统进程表是一项有限的资源，如果它被僵尸进程耗尽的话，系统可能会无法创建新的进程。


##### 如果是Zygote进程终止了，则会如何？

sigchld_handler_init()函数内部会找到Zygote进程并移除所有的Zygote进程的信息，在重启Zygote服务的启动脚本（如init.zygote64.rc）中带有onrestart选项的服务。


#### 4、启动属性服务（其中会启动servicemanager(binder服务大管家)、bootanim(开机动画)等重要服务）

    start_property_service();
    
    
##### 属性服务是如何启动的？

我们查看system/core/init/property_service.cpp源码中的start_property_service()函数：

    void start_property_service() {
        selinux_callback cb;
        cb.func_audit = SelinuxAuditCallback;
        selinux_set_callback(SELINUX_CB_AUDIT, cb);
    
        property_set("ro.property_service.version", "2");
    
        // 1
        property_set_fd = CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                       false, 0666, 0, 0,  nullptr);
        if (property_set_fd == -1) {
            PLOG(FATAL) << "start_property_service socket creation failed";
        }
    
        // 2
        listen(property_set_fd, 8);
    
        // 3、4、5
        register_epoll_handler(property_set_fd, handle_property_set_fd);
    }


- 1、首先，创建非阻塞式的Socket，并返回property_set_fd文件描述符。
- 2、使用listen()函数去监听property_set_fd，此时Socket即成为属性服务端，并且它最多同时可为8个试图设置属性的用户提供服务。
- 3、使用epoll()来监听property_set_fd：当property_set_fd中有数据到来时，init进程将调用handle_property_set_fd()函数进行处理。在Andorid 8.0的源码中则在handle_property_set_fd()函数中添加了handle_property_set函数做进一步封装处理。
- 4、系统属性分为两种属性，即普通属性和控制属性。控制属性用来执行一些命令，比如开机的动画就使用了这种属性。在handle_property_set_fd()函数中会先判断如果属性名是以"ctl."开头的，就说明是控制属性，如果客户端权限满足，则会调用handle_control_message()函数来修改控制属性。如果是普通属性，则会在客户端全面满足的条件下调用property_set函数来修改普通属性。
- 5、在property_set中会先从属性存储空间中查找该属性，如果有，则更新，否则添加该属性。此外，如果名称是以"ro"开头（表示只读，不能修改），直接返回，如果名称是以"persist."开头，则写入持久化属性。


##### epoll是什么？

在Linux的新内核中，epoll是用来取代select/poll的，它是Linux内核为处理大批量文件描述符的改进版poll，是Linux下多路复用I/O接口select/poll的增强版，它能显著提升程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。


##### epoll和select的区别？

epoll内部用于保存事件的数据类型是红黑树，查找速度快，select采用的数组保存信息，查找速度很慢，只有当等待少量文件描述符时，epoll和select的效率才差不多。


#### 5、解析init.rc配置文件

    parser.ParseConfig("/init.rc");
    

##### init.rc是什么？

它是由Android初始化语言编写的一个非常重要的配置脚本文件。Android初始化语言主要包含5种类型的语句：

- Action（常用）
- Service（常用）
- Command
- Option
- Import

这里了解下Action和Service的格式：

    on <trigger> [&& <trigger>]*     //设置触发器  
        <command>  
        <command>      //动作触发之后要执行的命令
        ...


    service <name> <pathname> [ <argument> ]*   //<service的名字><执行程序路径><传递参数>  
        <option>       //option是service的修饰词，影响什么时候、如何启动services  
        <option>  
        ...


注意：Android8.0对init.rc文件进行了拆分，每个服务对应一个rc文件。


##### init.rc中的Action、Service语句都有相应的XXXParser类来解析，即ActionParser、ServiceParser。那么ServiceParser是如何解析Service语句的？

查看相应的XXXparser，即ServiceParser：

    bool ServiceParser::ParseSection(const std::vector<std::string>& args,
                                     std::string* err) {
        
        // 判断Service是否有name和可执行程序                             
        if (args.size() < 3) {
            *err = "services must have a name and a     program";
            return false;
        }
        
        const std::string& name = args[1];
        if (!IsValidName(name)) {
            *err = StringPrintf("invalid service name     '%s'", name.c_str());
            return false;
        }
        
        std::vector<std::string> str_args(args.begin() + 2, args.end());
        // 1
        service_ = std::make_unique<Service>(name, str_args);
        return true;
    }
    
    
    bool ServiceParser::ParseLineSection(const std::vector<std::string>& args,
                                         const std::string&  filename, int line,
                                         std::string* err)  const {
        // 2                                 
        return service_ ? service_->ParseLine(args, err) : false;
    }
    
    void ServiceParser::EndSection() {
        if (service_) {
            // 3
            ServiceManager::GetInstance().AddService(std::move(service_));
        }
    }
    
    void ServiceManager::AddService(std::unique_ptr<Service> service) {
        Service* old_service = FindServiceByName(service->name());
        if (old_service) {
            ERROR("ignored duplicate definition of service '%s'",
                service->name().c_str());
            return;
        }
        services_.emplace_back(std::move(service));//1
    }
    

- 1、先使用ParseSection()方法根据参数创建出一个Service对象。
- 2、再使用ParseLineSection()方法解析Service语句中的每一个子项，将其中的内容添加到Service对象中。
- 3、然后，在解析完所有数据后，会调用EndSection函数，内部会执行ServiceManager的AddService函数，最终将Service对象加入vector类型的Service链表中。


##### init启动Zygote流程？

先看到init.rc的这部分配置代码：

    ...
    on nonencrypted    
        exec - root -- /system/bin/update_verifier nonencrypted  
        // 1
        class_start main         
        class_start late_start 
    ...


1、使用class_start这个COMMAND去启动Zygote。其中class_start对应do_class_start()函数。


    static Result<Success> do_class_start(const BuiltinArguments& args) {
        // Starting a class does not start services which are explicitly disabled.
        // They must be started individually.
        for (const auto& service : ServiceList::GetInstance()) {
            if (service->classnames().count(args[1])) {
                // 2
                if (auto result = service->StartIfNotDisabled(); !result) {
                    LOG(ERROR) << "Could not start service'" << service->name()
                               << "' as part of class '" <<  args[1] << "': " <<  result.error();
                }
            }
        }
        return Success();
    }


2、在system/core/init/builtins.cpp的do_class_start()函数中会遍历前面的Vector类型的Service链表，找到classname为main的Zygote，并调用system/core/init/service.cpp中的startIfNotDisabled()函数。


    bool Service::StartIfNotDisabled() {
        if (!(flags_ & SVC_DISABLED)) {
            return Start();
        } else {
            flags_ |= SVC_DISABLED_START;
        }
        return Success();
    }


3、如果Service没有再其对应的rc文件中设置disabled选项，则会调用Start()启动该Service。


    bool Service::Start() {
    
        ...
        
        
        if (flags_ & SVC_RUNNING) {
            if ((flags_ & SVC_ONESHOT) && disabled) {
                flags_ |= SVC_RESTART;
            }
            // 如果不是一个错误，尝试去启动一个已经启动的服务
            return Success();
        }
        
        ...
    
        // 判断需要启动的Service的对应的执行文件是否存在，不存在则不启动该Service
        struct stat sb;
        if (stat(args_[0].c_str(), &sb) == -1) {
            flags_ |= SVC_DISABLED;
            return ErrnoError() << "Cannot find '" << args_[0] << "'";
        }

        ...
        
        // fork函数创建子进程
        pid_t pid = fork();
        // 运行在子进程中
        if (pid == 0) {
            umask(077);
            ...
        
            // 在ExpandArgsAndExecv函数里进行了参数装配并使用了execve()执行程序
            if (!ExpandArgsAndExecv(args_)) {
                PLOG(ERROR) << "cannot execve('" << args_[0] << "')";
            }

             _exit(127);
        }
        ...
        return true;
    }
    
    static bool ExpandArgsAndExecv(cons std::vector<std::string>& args) {
        std::vector<std::string> expanded_args;
        std::vector<char*> c_strings;
    
        expanded_args.resize(args.size());
        c_strings.push_back(const_cast<char*>(args[0].data()));
        for (std::size_t i = 1; i < args.size(); ++i) {
            if (!expand_props(args[i], &expanded_args[i])) {
                LOG(FATAL) << args[0] << ": cannot expand '" << args[i] << "'";
            }
            c_strings.push_back(expanded_args[i].data());
        }
        c_strings.push_back(nullptr);
    
        // 最终通过execve执行程序
        return execv(c_strings[0], c_strings.data()) == 0;
    }


4、在Start()函数中，如果Service已经运行，则不再启动。如果没有，则使用fork()函数创建子进程，并返回pid值。当pid为0时，则说明当前代码逻辑在子进程中运行，最然后会调用execve()函数去启动子进程，并进入该Service的main函数中，如果该Service是Zygote，则会执行Zygote的main函数。（对应frameworks/base/cmds/app_process/app_main.cpp中的main()函数）


    int main(int argc, char* const argv[])
    {
        ...
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


5、最后，调用runtime的start函数启动Zygote。
    

### 五、总结

经过以上的分析，init进程的启动过程主要分为以下三部：

- 1、创建和挂载启动所需的文件目录。
- 2、初始化和启动属性服务。
- 3、解析init.rc配置文件并启动Zygote进程。

下篇，将会继续为大家讲解Android系统启动流程中的Zygote进程和SystemService启动过程，敬请期待~


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

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。