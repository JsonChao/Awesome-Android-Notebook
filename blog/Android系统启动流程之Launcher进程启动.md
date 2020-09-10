---

		title:  Android系统启动流程之Launcher进程启动
		date: 2019/3/9 15:00:00   
		tags: 
		- Android核心源码分析
		categories: Android核心源码分析
		thumbnail: http://images.apusapps.com/src/launcher-themes.jpg
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

前面几篇文章我们已经详细分析了Android系统启动流程的init进程、Zygote进程和SystemServer进程。本篇，我们来分析一下Launcher的启动过程。

Android系统启动的最后一步就是启动了一个Launcher应用程序来显示系统中已经安装的应用程序。**Launcher在启动的过程中会请求请求PMS返回系统中已安装的应用程序的信息，并将这些信息封装成一个快捷图标列表显示在系统屏幕上，从而使得用户可以点击这些快捷图片来启动相应的应用程序。**

Launcher作为Android系统的桌面，它的作用有两点：

- 1、作为Android系统的启动器，用于启动应用程序。
- 2、作为Android系统的桌面，用于显示和管理应用程序的快捷图标或者其它桌面组件。

下面，我们从以下两个方面来分析Launcher的启动过程：

- 1、Launcher启动流程
- 2、Launcher中应用图标的显示过程

### 一、Launcher启动过程分析

SystemServer进程在启动的过程中会启动PMS，PMS启动后会将系统中的应用程序安装完成，先前已经启动的AMS会将Launcher启动起来。在SystemServer的startOtherServices()方法中，调用了AMS的systemReady()方法，此即为Launcher的入口，如下所示：


    private void startOtherServices() {
        ...
        
        mActivityManagerService.systemReady(() -> {
            Slog.i(TAG, "Making services ready");
            traceBeginAndSlog("StartActivityManagerReadyPhase");
            mSystemServiceManager.startBootPhase(
                    SystemService.PHASE_ACTIVITY_MANAGER_READY);
                    
            ...
            }
        ...
    }
    

在Android 8.0及以上的部分源码中，都引入了Java Lambda表达式，可见其重要性的上升。下面继续分析AMS的systemReady()方法：


    public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
        ...
        
        synchronized (this) {
            ...
            mStackSupervisor.resumeFocusedStackTopActivityLocked();
            mUserController.sendUserSwitchBroadcasts(-1, currentUserId);
            
            ...
        }
    }
    

在systemReady()方法中继续调用了ActivityStackSupervisor的resumeFocusedStackTopActivityLocked()方法，如下所示：


    boolean resumeFocusedStackTopActivityLocked() {
        return resumeFocusedStackTopActivityLocked(null, null, null);
    }

    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        ...
        
        if (targetStack != null && isFocusedStack(targetStack)) {
            // 1
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        ...
        
        return false;
    }
    

最终，调用了注释1处ActivityStack（描述Acitivity堆栈）的resumeTopActivityUncheckedLocked()方法，如下所示：


    @GuardedBy("mService")
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            // 1
            result = resumeTopActivityInnerLocked(prev, options);
            
            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }

         return result;
    }
        

在注释1处调用了resumeTopActivityInnerLocked()方法，如下所示：


    @GuardedBy("mService")
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ...
        
         if (!hasRunningActivity) {
            // There are no activities left in the stack, let's look somewhere else.
            return resumeTopActivityInNextFocusableStack(prev, options, "noMoreActivities");
        }
        
        ...
    }
    
    
resumeTopActivityInnerLocked()方法非常长，大概有好几百行代码，但是对于主要流程来说最关键的就是在其中调用了resumeTopActivityInNextFocusableStack()方法，如下所示：


    private boolean resumeTopActivityInNextFocusableStack(ActivityRecord prev,
            ActivityOptions options, String reason) {
        if (adjustFocusToNextFocusableStack(reason)) {
            // Try to move focus to the next visible stack with a running activity if this
            // stack is not covering the entire screen or is on a secondary display (with no home
            // stack).
            return mStackSupervisor.resumeFocusedStackTopActivityLocked(
                    mStackSupervisor.getFocusedStack(), prev, null);
        }

        // Let's just start up the Launcher...
        ActivityOptions.abort(options);
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityInNextFocusableStack: " + reason + ", go home");
        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        // Only resume home if on home display
        // 1
        return isOnHomeDisplay() &&
                mStackSupervisor.resumeHomeStackTask(prev, reason);
    }
    
    
在注释1处，调用了ActivityStackSupervisor的resumeHomeStackTask()方法，如下所示：


    boolean resumeHomeStackTask(ActivityRecord prev, String reason) {
        ...

        // Only resume home activity if isn't finishing.
        if (r != null && !r.finishing) {
            moveFocusableActivityStackToFrontLocked(r, myReason);
            return resumeFocusedStackTopActivityLocked(mHomeStack, prev, null);
        }
        // 1
        return mService.startHomeActivityLocked(mCurrentUser, myReason);
    }


注释1处，调用了AMS的startHomeActivityLocked()方法，如下所示：


    boolean startHomeActivityLocked(int userId, String reason) {
        // 1
        if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
                && mTopAction == null) {
            // We are running in factory test mode, but unable to find
            // the factory test app, so just sit around displaying the
            // error message and don't try to start anything.
            return false;
        }
        
        // 2
        Intent intent = getHomeIntent();
        ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
            // Don't do this if the home app is currently being
            // instrumented.
            aInfo = new ActivityInfo(aInfo);
            aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
            ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                    aInfo.applicationInfo.uid, true);
                    
            // 3
            if (app == null || app.instr == null) {
                intent.setFlags(intent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
                final int resolvedUserId = UserHandle.getUserId(aInfo.applicationInfo.uid);
                // For ANR debugging to verify if the user activity is the one that actually
                // launched.
                final String myReason = reason + ":" + userId + ":" + resolvedUserId;
               
                // 4
                mActivityStartController.startHomeActivity(intent, aInfo, myReason);
            }
        } else {
            Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
        }

        return true;
    }


首先，会在注释1处判断工厂模式和mTopAction的值，这里的工厂模式mFactoryTest代表的了系统的运行模式，它分为三种：

- 1、非工厂模式
- 2、低级工厂模式
- 3、高级工厂模式

而mTopAction是来描述第一个被启动Activity组件的Action，默认值为Intent.ACTION_MAIN。所以，此时可知当mFactoryTest为低级工厂模式并且mTopAction为空时，则返回false。接着，在注释2处，调用了getHomeintent()方法，如下所示：


    Intent getHomeIntent() {
        // 1
        Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
        intent.setComponent(mTopComponent);
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
        if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            // 2
            intent.addCategory(Intent.CATEGORY_HOME);
        }
        return intent;
    }
    
    
在getHomeIntent()方法的注释1处，根据mTopAction和mTopData创建了Intent。注释2处，会判断如果系统运行模式不是低级工厂模式，则会将Category设置为Intent.CATEGORY_HOME，最后返回该Intent。

我们再回到AMS的startHomeActivityLocked()方法的注释3处，这里会判断符合上述Intent的应用程序是否已经启动，如果没有启动，则会在注释4处调用ActivityStartController的startHomeActivity()方法启动该应用程序，即Launcher。下面我们继续看看startHomeActivity()方法，如下所示：


    void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason) {
        // 1
        mSupervisor.moveHomeStackTaskToTop(reason);

        // 2
        mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
                .setOutActivity(tmpOutRecord)
                .setCallingUid(0)
                .setActivityInfo(aInfo)
                .execute();
        mLastHomeActivityStartRecord = tmpOutRecord[0];
        if (mSupervisor.inResumeTopActivity) {
            // If we are in resume section already, home activity will be initialized, but not
            // resumed (to avoid recursive resume) and will stay that way until something pokes it
            // again. We need to schedule another resume.
            mSupervisor.scheduleResumeTopActivities();
        }
    }


注释1处，会将Launcher放入HomeStack中，它是ActivityStackSupervisor中用于存储Launcher的变量。然后，在注释2处调用了obtainStarter()方法，如下所示：


    **
     * @return A starter to configure and execute starting an activity. It is valid until after
     *         {@link ActivityStarter#execute} is invoked. At that point, the starter should be
     *         considered invalid and no longer modified or used.
     */
    ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }


可知这里最终会返回一个配置好指定intent和reason和ActivityStarter，当它调用execute()方法时，则会启动Launcher，如下所示：


    int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            }
        } finally {
            onExecutionComplete();
        }
    }
    

可以看到，这里调用了startActivity()方法来启动Launcher，最终会进入Launcher的onCreate()方法，Launcher启动完成。


### 二、Launcher中应用图标的显示过程

应用程序图标是进入应用程序的入口，接下来我们了解一下Launcher是如何显示应用程序图标的。首先从Launcher的onCreate()方法开始，如下所示：


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        
        // 1
        LauncherAppState app = LauncherAppState.getInstance(this);
        mOldConfig = new Configuration(getResources().getConfiguration());
        
        // 2
        mModel = app.setLauncher(this);
        initDeviceProfile(app.getInvariantDeviceProfile());
        
        ...
        
        // We only load the page synchronously if the user rotates (or triggers a
        // configuration change) while launcher is in the foreground
        int currentScreen = PagedView.INVALID_RESTORE_PAGE;
        if (savedInstanceState != null) {
            currentScreen = savedInstanceState.getInt(RUNTIME_STATE_CURRENT_SCREEN, currentScreen);
        }
        
        // 3
        if (!mModel.startLoader(currentScreen)) {
            if (!internalStateHandled) {
                // If we are not binding synchronously, show a fade in animation when
                // the first page bind completes.
                mDragLayer.getAlphaProperty(ALPHA_INDEX_LAUNCHER_LOAD).setValue(0);
            }
        } else {
            // Pages bound synchronously.
            mWorkspace.setCurrentPage(currentScreen);

            setWorkspaceLoading(true);
        }
    
    }


首先，在注释1处得到LauncherAppState的实例，在注释2处，调用了它的setLauncher()方法将Launcher对象传进去，setLauncher()方法如下所示：


     LauncherModel setLauncher(Launcher launcher) {
        getLocalProvider(mContext).setLauncherProviderChangeListener(launcher);
        mModel.initialize(launcher);
        return mModel;
    }


在setLauncher()方法里面继续调用了LauncherModel的initialize()方法，如下所示：

    
    /**
    * Set this as the current Launcher activity object for the loader.
    */
    public void initialize(Callbacks callbacks) {
        synchronized (mLock) {
            Preconditions.assertUIThread();
            mCallbacks = new WeakReference<>(callbacks);
        }
    }


从此处我们可以得知Launcher被封装成了一个弱引用对象mCallbacks。我们再回到Launcher的onCreate()方法的注释3处的LauncherModel的startLoader()方法，如下所示：


    // 1
    @Thunk static final HandlerThread sWorkerThread = new HandlerThread("launcher-loader");
    static {
        sWorkerThread.start();
    }
    
    // 2
    @Thunk static final Handler sWorker = new Handler(sWorkerThread.getLooper());
    
    public boolean startLoader(int synchronousBindPage) {
        // Enable queue before starting loader. It will get disabled in Launcher#finishBindingItems
        InstallShortcutReceiver.enableInstallQueue(InstallShortcutReceiver.FLAG_LOADER_RUNNING);
        synchronized (mLock) {
            // Don't bother to start the thread if we know it's not going to do anything
            if (mCallbacks != null && mCallbacks.get() != null) {
                final Callbacks oldCallbacks = mCallbacks.get();
                // Clear any pending bind-runnables from the synchronized load process.
                mUiExecutor.execute(oldCallbacks::clearPendingBinds);

                // If there is already one running, tell it to stop.
                stopLoader();
                
                // 3
                LoaderResults loaderResults = new LoaderResults(mApp, sBgDataModel,
                        mBgAllAppsList, synchronousBindPage, mCallbacks);
                if (mModelLoaded && !mIsLoaderTaskRunning) {
                    // Divide the set of loaded items into those that we are binding synchronously,
                    // and everything else that is to be bound normally (asynchronously).
                    loaderResults.bindWorkspace();
                    // For now, continue posting the binding of AllApps as there are other
                    // issues that arise from that.
                    loaderResults.bindAllApps();
                    loaderResults.bindDeepShortcuts();
                    loaderResults.bindWidgets();
                    return true;
                } else {
                    // 4
                    startLoaderForResults(loaderResults);
                }
            }
        }
        return false;
    }


在注释1处，新建了具有消息循环的线程HandlerThread对象。注释2处，新建了Handler，并传入了HandlerThread的Looper，此处Handler就是用于向HandlerThread发送消息。接着，在注释3处，创建了LoaderResults，在注释4处，调用了startLoaderForResults()方法并将LoaderResults传入，如下所示：


    public void startLoaderForResults(LoaderResults results) {
        synchronized (mLock) {
            stopLoader();
            mLoaderTask = new LoaderTask(mApp, mBgAllAppsList, sBgDataModel, results);
            runOnWorkerThread(mLoaderTask);
        }
    }


在startLoaderForResults()方法中，调用了runOnWorkerThread()，如下所示：


    /** Runs the specified runnable immediately if called from the worker thread, otherwise it is
     * posted on the worker thread handler. */
    private static void runOnWorkerThread(Runnable r) {
        // 1
        if (sWorkerThread.getThreadId() == Process.myTid()) {
            // 2
            r.run();
        } else {
            // If we are not on the worker thread, then post to the worker handler
            // 3
            sWorker.post(r);
        }
    }


首先，注释1处会先判断当前的执行线程是否是工作线程，如果是则直接调用注释2处Runnable的run()方法，否则，调用sWorker这个Handler对象的post()方法将LoaderTask作为消息发送给HandlerThread。接下来，我们看看LoaderTask，它实现了Runnable接口，当其所描述的消息被处理时，则会调用它的run()方法，如下所示：


    /**
    * Runnable for the thread that loads the contents of the launcher:
    *   - workspace icons
    *   - widgets
    *   - all apps icons
    *   - deep shortcuts within apps
    */
    public class LoaderTask implements Runnable {
    
        ...
        
         synchronized (this) {
            // Skip fast if we are already stopped.
            if (mStopped) {
                return;
            }
        }

        TraceHelper.beginSection(TAG);
        try (LauncherModel.LoaderTransaction transaction = mApp.getModel().beginLoader(this)) {
            TraceHelper.partitionSection(TAG, "step 1.1: loading workspace");
            // 1
            loadWorkspace();

            verifyNotStopped();
            TraceHelper.partitionSection(TAG, "step 1.2: bind workspace workspace");
            // 2
            mResults.bindWorkspace();

            // Notify the installer packages of packages with active installs on the first screen.
            TraceHelper.partitionSection(TAG, "step 1.3: send first screen broadcast");
            sendFirstScreenActiveInstallsBroadcast();

            // Take a break
            TraceHelper.partitionSection(TAG, "step 1 completed, wait for idle");
            waitForIdle();
            verifyNotStopped();

            // second step
            TraceHelper.partitionSection(TAG, "step 2.1: loading all apps");
            // 3
            loadAllApps();
            
            
            TraceHelper.partitionSection(TAG, "step 2.2: Binding all apps");
            verifyNotStopped();
            // 4
            mResults.bindAllApps();
            
            ...
         } catch (CancellationException e) {
            // Loader stopped, ignore
            TraceHelper.partitionSection(TAG, "Cancelled");
        }
        TraceHelper.endSection(TAG);
    }
    
    
**Launcher是用工作区的形式来显示系统安装的应用程序快捷图标的，每一个工作区都是用来描述一个抽象桌面的，它由n个屏幕组成，每个屏幕又分为n个单元格，每个单元格用来显示一个应用程序的快捷图标。**

首先，在注释1、2处调用了loadWorkSpace()和LoaderResults的bindWorkspace()方法来加载和绑定工作区信息。注释3处调用了loadAllApps()和LoaderResults的bindAllApps()方法来加载系统已经安装的应用程序信息，bindAllApps()方法如下所示：


    public void bindAllApps() {
        // shallow copy
        @SuppressWarnings("unchecked")
        final ArrayList<AppInfo> list = (ArrayList<AppInfo>) mBgAllAppsList.data.clone();

        Runnable r = new Runnable() {
            public void run() {
                // 1
                Callbacks callbacks = mCallbacks.get();
                if (callbacks != null) {
                    // 2
                    callbacks.bindAllApplications(list);
                }
            }
        };
        // 3
        mUiExecutor.execute(r);
    }
    
    
首先，在注释1处会从mCallbacks这个Launcher的弱引用对象中取出Launcher对象，并在注释2处调用了它的bindAllApplication()来绑定所有的应用程序信息，最后在注释3处使用mUiExecutor这个MainThreadExecutor执行器对象去执行这个创建好的Runnable对象。接下来，我们看看Launcher的bindAllApplications()方法，如下所示：

    // Main container view for the all apps screen.
    @Thunk AllAppsContainerView mAppsView;

    /**
    * Add the icons for all apps.
    *
    * Implementation of the method from LauncherModel.Callbacks.
    */
    public void bindAllApplications(ArrayList<AppInfo> apps) {
        // 1
        mAppsView.getAppsStore().setApps(apps);

        if (mLauncherCallbacks != null) {
            mLauncherCallbacks.bindAllApplications(apps);
        }
    }

在注释1处，调用了AllAppsContainerView的getAppsStore()方法得到了一个AllAppsStore对象，AllAppsContainerView是所有App屏幕的主容器视图，AllAppsStore是一个负责维护所有app信息集合的通用工具类。下面，我们看看AllAppsStore对象的setApps()方法：


    /**
     * Sets the current set of apps.
     */
    public void setApps(List<AppInfo> apps) {
        mComponentToAppMap.clear();
        addOrUpdateApps(apps);
    }

    
这里继续调用了addOrUpdateApps()方法：

     private final HashMap<ComponentKey, AppInfo> mComponentToAppMap = new HashMap<>();

    /**
    * Adds or updates existing apps in the list
    */
    public void addOrUpdateApps(List<AppInfo> apps) {
        for (AppInfo app : apps) {
            mComponentToAppMap.put(app.toComponentKey(), app);
        }
        notifyUpdate();
    }

    
可以看到，最终将所有app信息保存在了AllAppsStore的HashMap容器中。

当AllAppsContainerView加载完XML布局时，会调用自身的onFinishInflate()方法，如下所示：


    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();

        // This is a focus listener that proxies focus from a view into the list view.  This is to
        // work around the search box from getting first focus and showing the cursor.
        setOnFocusChangeListener((v, hasFocus) -> {
            if (hasFocus && getActiveRecyclerView() != null) {
                getActiveRecyclerView().requestFocus();
            }
        });

        mHeader = findViewById(R.id.all_apps_header);
        
        // 1
        rebindAdapters(mUsingTabs, true /* force */);

        mSearchContainer = findViewById(R.id.search_container_all_apps);
        mSearchUiManager = (SearchUiManager) mSearchContainer;
        mSearchUiManager.initialize(this);
    }
    
    
在注释1处，进行了适配器数据的绑定，我们继续查看rebindAdapters()方法：


    private void rebindAdapters(boolean showTabs) {
        rebindAdapters(showTabs, false /* force */);
    }

    private void rebindAdapters(boolean showTabs, boolean force) {
        ...

        if (mUsingTabs) {
            // 1
            mAH[AdapterHolder.MAIN].setup(mViewPager.getChildAt(0), mPersonalMatcher);
            mAH[AdapterHolder.WORK].setup(mViewPager.getChildAt(1), mWorkMatcher);
            onTabChanged(mViewPager.getNextPage());
        } else {
            // 2
            mAH[AdapterHolder.MAIN].setup(findViewById(R.id.apps_list_view), null);
            mAH[AdapterHolder.WORK].recyclerView = null;
        }
        setupHeader();

        ...
    }


可以看到，不管是否正在使用标签，最终都会调用到AdapterHolder的setup()方法，它时AllAppsContainerView的内部类，如下所示：


    void setup(@NonNull View rv, @Nullable ItemInfoMatcher matcher) {
            appsList.updateItemFilter(matcher);
            recyclerView = (AllAppsRecyclerView) rv;
            recyclerView.setEdgeEffectFactory(createEdgeEffectFactory());
            
            // 1
            recyclerView.setApps(appsList, mUsingTabs);
            recyclerView.setLayoutManager(layoutManager);
            
            // 2
            recyclerView.setAdapter(adapter);
            recyclerView.setHasFixedSize(true);
            // No animations will occur when changes occur to the items in this RecyclerView.
            recyclerView.setItemAnimator(null);
            FocusedItemDecorator focusedItemDecorator = new FocusedItemDecorator(recyclerView);
            recyclerView.addItemDecoration(focusedItemDecorator);
            adapter.setIconFocusListener(focusedItemDecorator.getFocusListener());
            applyVerticalFadingEdgeEnabled(verticalFadingEdge);
            applyPadding();
        }
        
        
注释1处，会将app信息列表appsList设置给AllAppsRecyclerView对象，在注释2处，为其设置了Adapter。最终，应用程序快捷图标列表就会显示到屏幕上了。
    
    
### 三、总结

到此，我们终于将Android系统启动流程这一主题分析完毕，结合前面的几篇内容，可以得出核心流程如下：

- 1、**启动电源以及系统启动**：当电源按下时引导芯片从预定义的订房（固化在ROM）开始执行，加载引导程序BootLoader到RAM，然后执行。
- 2、**引导程序BootLoader**：BootLoader是在Android系统开始运行前的一个小程序，主要用于把系统OS拉起来并运行。。
- 3、**Linux内核启动**：当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当其完成系统设置时，会先在系统文件中寻找init.rc文件，并启动init进行。
- 4、**init进程启动**：初始化和启动属性服务，并且启动Zygote进程。
- 5、**Zygote进程启动**：创建JVM并为其注册JNI方法，创建服务器端Socket，启动SystemServer进程。
- 6、**SystemServer进程启动**：启动Binder线程池和SystemServiceManager，并且启动各种系统服务。
- 7、**Launcher启动**：被SystemServer进程启动的AMS会启动Launcher，Launcher启动后会将已安装应用的快捷图标显示到系统桌面上。

下一系列，笔者将会给大家带来Android中的跨进程通信Binder的详细讲解，尽请期待~


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