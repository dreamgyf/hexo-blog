---
title: Android源码分析-ActivityManagerService
date: 2022-07-26 18:03:08
tags: 
- Android源码
- ActivityManagerService
categories: 
- [Android, 源码分析]
- [Android, ActivityManagerService]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

作为一名`Android`开发，我们最熟悉并且最常打交道的当然非四大组件中的`Activity`莫属，这次我们就来讲讲`ActivityManagerService`这个`Android`系统核心服务究竟扮演了一个怎样的角色，提供了哪些功能

# 简介

`ActivityManagerService`（以下简称`AMS`）是`Android`系统中最核心的服务之一，主要负责系统中四大组件的调度管理

# 启动

我们在[Android源码分析 - SystemServer（下）](https://juejin.cn/post/7058543470356463646)中提到过`AMS`的启动，在`SystemServer`启动的各个阶段，`AMS`做了一些不同的工作，我们先从它的实例化开始

```java
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
    ...
    //创建 ATMS & AMS
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    ...
}
```

## ActivityTaskManagerService

路径：`frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java`

`Android 10`以后增加了一个`ActivityTaskManagerService`（以下简称`ATMS`）来分担`AMS`的工作

### startService

```java
public <T extends SystemService> T startService(Class<T> serviceClass) {
    try {
        final String name = serviceClass.getName();
        //必须是SystemService的子类
        if (!SystemService.class.isAssignableFrom(serviceClass)) {
            throw new RuntimeException(...);
        }
        final T service;
        //反射调用构造方法实例化
        try {
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        } catch (...) {
            ...
        }

        startService(service);
        return service;
    } finally {
        ...
    }
}

public void startService(@NonNull final SystemService service) {
    //添加到List中
    mServices.add(service);
    
    long time = SystemClock.elapsedRealtime();
    try {
        //回调
        service.onStart();
    } catch (RuntimeException ex) {
        throw new RuntimeException(...);
    }
    //启动时间过长会打一个Warnning级别日志
    warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
}
```

`ATMS`和`AMS`的启动通过了`SystemServiceManager.startService`方法，它需要参数为`SystemService`子类的`class`，对于`ATMS`来说，它的内部类`Lifecycle`继承自`SystemService`

`SystemServiceManager.startService`方法首先通过反射创建了对应传进来类的实例，然后将其添加到`SystemServiceManager`内部的一个`List`中，接着回调其`onStart`方法

```java
public static final class Lifecycle extends SystemService {
    private final ActivityTaskManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityTaskManagerService(context);
    }

    @Override
    public void onStart() {
        publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
        mService.start();
    }

    @Override
    public void onUnlockUser(int userId) {
        synchronized (mService.getGlobalLock()) {
            mService.mStackSupervisor.onUserUnlocked(userId);
        }
    }

    @Override
    public void onCleanupUser(int userId) {
        synchronized (mService.getGlobalLock()) {
            mService.mStackSupervisor.mLaunchParamsPersister.onCleanupUser(userId);
        }
    }

    public ActivityTaskManagerService getService() {
        return mService;
    }
}
```

其实`ATMS.Lifecycle`很短，我们看它的构造方法，其实就是实例化了一个`ATMS`出来

```java
public ActivityTaskManagerService(Context context) {
    mContext = context;
    mFactoryTest = FactoryTest.getMode();
    mSystemThread = ActivityThread.currentActivityThread();
    mUiContext = mSystemThread.getSystemUiContext();
    //管理Activity生命周期
    mLifecycleManager = new ClientLifecycleManager();
    mInternal = new LocalService();
    GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version", GL_ES_VERSION_UNDEFINED);
    mWindowOrganizerController = new WindowOrganizerController(this);
    mTaskOrganizerController = mWindowOrganizerController.mTaskOrganizerController;
}
```

这里面初始化了很多东西，我们暂且跳过，等以后使用到时候在细说，然后`startService`方法会回调`ATMS.Lifecycle.onStart`方法，这个方法首先将创建好的`ATMS`注册到`ServiceManager`中

```java
protected final void publishBinderService(@NonNull String name, @NonNull IBinder service) {
    publishBinderService(name, service, false);
}

protected final void publishBinderService(@NonNull String name, @NonNull IBinder service,
        boolean allowIsolated) {
    publishBinderService(name, service, allowIsolated, DUMP_FLAG_PRIORITY_DEFAULT);
}

protected final void publishBinderService(String name, IBinder service,
        boolean allowIsolated, int dumpPriority) {
    ServiceManager.addService(name, service, allowIsolated, dumpPriority);
}
```

然后调用了`ATMS.start`方法，将刚创建出来的`mInternal`添加到本地服务中

```java
private void start() {
    LocalServices.addService(ActivityTaskManagerInternal.class, mInternal);
}
```

## ActivityManagerService

接着我们回到`SystemServer.startBootstrapServices`中，接下来是对`AMS`的启动

### startService

`AMS`的启动和`ATMS`类似，它们同样是通过内部的一个`Lifecycle`类

```java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;
    private static ActivityTaskManagerService sAtm;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context, sAtm);
    }

    public static ActivityManagerService startService(
            SystemServiceManager ssm, ActivityTaskManagerService atm) {
        sAtm = atm;
        return ssm.startService(ActivityManagerService.Lifecycle.class).getService();
    }

    @Override
    public void onStart() {
        mService.start();
    }

    @Override
    public void onBootPhase(int phase) {
        mService.mBootPhase = phase;
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            mService.mBatteryStatsService.systemServicesReady();
            mService.mServices.systemServicesReady();
        } else if (phase == PHASE_ACTIVITY_MANAGER_READY) {
            mService.startBroadcastObservers();
        } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
            mService.mPackageWatchdog.onPackagesReady();
        }
    }

    @Override
    public void onUserStopped(@NonNull TargetUser user) {
        mService.mBatteryStatsService.onCleanupUser(user.getUserIdentifier());
    }

    public ActivityManagerService getService() {
        return mService;
    }
}
```

可以看到，这里将`ATMS`作为参数参与`AMS`的实例化，我么看一下它的构造方法

```java
public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
    LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
    mInjector = new Injector(systemContext);
    mContext = systemContext;

    mFactoryTest = FactoryTest.getMode();
    //获取系统ActivityThread
    mSystemThread = ActivityThread.currentActivityThread();
    mUiContext = mSystemThread.getSystemUiContext();

    Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());

    //创建线程来处理AMS的各种状态
    mHandlerThread = new ServiceThread(TAG,
            THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
    mHandlerThread.start();
    mHandler = new MainHandler(mHandlerThread.getLooper());
    mUiHandler = mInjector.getUiHandler(this);

    //创建proc线程
    mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
            THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
    mProcStartHandlerThread.start();
    mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());

    //获取定义常量
    mConstants = new ActivityManagerConstants(mContext, this, mHandler);
    //记录活跃的进程uid
    final ActiveUids activeUids = new ActiveUids(this, true /* postChangesToAtm */);
    mPlatformCompat = (PlatformCompat) ServiceManager.getService(
            Context.PLATFORM_COMPAT_SERVICE);
    //新建ProcessList对象用来处理进程
    mProcessList = mInjector.getProcessList(this);
    mProcessList.init(this, activeUids, mPlatformCompat);
    //低内存检测器
    mLowMemDetector = new LowMemDetector(this);
    //OOM调节器
    mOomAdjuster = new OomAdjuster(this, mProcessList, activeUids);

    //定义广播策略参数
    final BroadcastConstants foreConstants = new BroadcastConstants(
            Settings.Global.BROADCAST_FG_CONSTANTS);
    foreConstants.TIMEOUT = BROADCAST_FG_TIMEOUT;

    final BroadcastConstants backConstants = new BroadcastConstants(
            Settings.Global.BROADCAST_BG_CONSTANTS);
    backConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;

    final BroadcastConstants offloadConstants = new BroadcastConstants(
            Settings.Global.BROADCAST_OFFLOAD_CONSTANTS);
    offloadConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
    // by default, no "slow" policy in this queue
    offloadConstants.SLOW_TIME = Integer.MAX_VALUE;

    mEnableOffloadQueue = SystemProperties.getBoolean(
            "persist.device_config.activity_manager_native_boot.offload_queue_enabled", false);

    //初始化前台广播队列
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "foreground", foreConstants, false);
    //初始化后台广播队列
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "background", backConstants, true);
    //初始化卸载广播队列
    mOffloadBroadcastQueue = new BroadcastQueue(this, mHandler,
            "offload", offloadConstants, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;
    mBroadcastQueues[2] = mOffloadBroadcastQueue;

    mServices = new ActiveServices(this);
    mProviderMap = new ProviderMap(this);
    mPackageWatchdog = PackageWatchdog.getInstance(mUiContext);
    mAppErrors = new AppErrors(mUiContext, this, mPackageWatchdog);

    final File systemDir = SystemServiceManager.ensureSystemDir();

    // TODO: Move creation of battery stats service outside of activity manager service.
    mBatteryStatsService = new BatteryStatsService(systemContext, systemDir,
            BackgroundThread.get().getHandler());
    mBatteryStatsService.getActiveStatistics().readLocked();
    mBatteryStatsService.scheduleWriteToDisk();
    mOnBattery = DEBUG_POWER ? true
            : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
    mBatteryStatsService.getActiveStatistics().setCallback(this);
    mOomAdjProfiler.batteryPowerChanged(mOnBattery);

    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

    mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);

    mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);

    mUserController = new UserController(this);

    mPendingIntentController = new PendingIntentController(
            mHandlerThread.getLooper(), mUserController, mConstants);

    if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
        mUseFifoUiScheduling = true;
    }

    mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
    mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);

    //ATMS进一步初始化
    mActivityTaskManager = atm;
    mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
            DisplayThread.get().getLooper());
    mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);

    mProcessCpuThread = new Thread("CpuTracker") {
        @Override
        public void run() {
            synchronized (mProcessCpuTracker) {
                mProcessCpuInitLatch.countDown();
                mProcessCpuTracker.init();
            }
            while (true) {
                try {
                    try {
                        synchronized(this) {
                            final long now = SystemClock.uptimeMillis();
                            long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                            long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                            //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                            //        + ", write delay=" + nextWriteDelay);
                            if (nextWriteDelay < nextCpuDelay) {
                                nextCpuDelay = nextWriteDelay;
                            }
                            if (nextCpuDelay > 0) {
                                mProcessCpuMutexFree.set(true);
                                this.wait(nextCpuDelay);
                            }
                        }
                    } catch (InterruptedException e) {
                    }
                    updateCpuStatsNow();
                } catch (Exception e) {
                    Slog.e(TAG, "Unexpected exception collecting process stats", e);
                }
            }
        }
    };

    mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);

    Watchdog.getInstance().addMonitor(this);
    Watchdog.getInstance().addThread(mHandler);

    // bind background threads to little cores
    // this is expected to fail inside of framework tests because apps can't touch cpusets directly
    // make sure we've already adjusted system_server's internal view of itself first
    updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
    try {
        Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(),
                Process.THREAD_GROUP_SYSTEM);
        Process.setThreadGroupAndCpuset(
                mOomAdjuster.mCachedAppOptimizer.mCachedAppOptimizerThread.getThreadId(),
                Process.THREAD_GROUP_SYSTEM);
    } catch (Exception e) {
        Slog.w(TAG, "Setting background thread cpuset failed");
    }

    mInternal = new LocalService();
    mPendingStartActivityUids = new PendingStartActivityUids(mContext);
}
```

这里的内容更复杂了，我们注意到`AMS`将`ATMS`保存成了自己的一个成员变量，然后调用它的`initialize`方法进一步初始化

```java
public void initialize(IntentFirewall intentFirewall, PendingIntentController intentController,
        Looper looper) {
    mH = new H(looper);
    mUiHandler = new UiHandler();
    mIntentFirewall = intentFirewall;
    final File systemDir = SystemServiceManager.ensureSystemDir();
    mAppWarnings = createAppWarnings(mUiContext, mH, mUiHandler, systemDir);
    mCompatModePackages = new CompatModePackages(this, systemDir, mH);
    mPendingIntentController = intentController;
    mStackSupervisor = createStackSupervisor();

    mTaskChangeNotificationController =
            new TaskChangeNotificationController(mGlobalLock, mStackSupervisor, mH);
    mLockTaskController = new LockTaskController(mContext, mStackSupervisor, mH);
    mActivityStartController = new ActivityStartController(this);
    setRecentTasks(new RecentTasks(this, mStackSupervisor));
    mVrController = new VrController(mGlobalLock);
    mKeyguardController = mStackSupervisor.getKeyguardController();
}
```

然后同样的，`startService`方法回调`AMS.Lifecycle.onStart`方法，调用`AMS.start`方法

```java
private void start() {
    removeAllProcessGroups();
    mProcessCpuThread.start();

    mBatteryStatsService.publish();
    mAppOpsService.publish();
    Slog.d("AppOps", "AppOpsService published");
    LocalServices.addService(ActivityManagerInternal.class, mInternal);
    //通知ATMS，ActivityManagerInternal被添加到了本地服务中，ATMS那边可以获取到了
    mActivityTaskManager.onActivityManagerInternalAdded();
    mPendingIntentController.onActivityManagerInternalAdded();
    // Wait for the synchronized block started in mProcessCpuThread,
    // so that any other access to mProcessCpuTracker from main thread
    // will be blocked during mProcessCpuTracker initialization.
    try {
        mProcessCpuInitLatch.await();
    } catch (InterruptedException e) {
        Slog.wtf(TAG, "Interrupted wait during start", e);
        Thread.currentThread().interrupt();
        throw new IllegalStateException("Interrupted wait during start");
    }
}
```

### 后续工作

后续在`SystemServer`启动服务的过程中，`AMS`还参加进行了很多工作，我在这里将其列出来

```java
/* 阶段0 */
mActivityManagerService = ActivityManagerService.Lifecycle.startService(
        mSystemServiceManager, atm);
mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
mActivityManagerService.setInstaller(installer);
mActivityManagerService.initPowerManagement();

/* 阶段100 PHASE_WAIT_FOR_DEFAULT_DISPLAY */

//注册各种系统服务
mActivityManagerService.setSystemProcess();
//应用用量统计服务
mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));
//加载SettingProvider
mActivityManagerService.installSystemProviders();
//窗口服务
mActivityManagerService.setWindowManager(wm);
//进入安全模式
if (safeMode) {
    mActivityManagerService.enterSafeMode();
}

/* 阶段480 PHASE_LOCK_SETTINGS_READY */

/* 阶段500 PHASE_SYSTEM_SERVICES_READY */

mActivityManagerService.systemReady(() -> {
    /* 阶段550 PHASE_ACTIVITY_MANAGER_READY */
    ...
    /* 阶段600 PHASE_THIRD_PARTY_APPS_CAN_START */
    ...
}, t);
```

我们拣几个比较重要的方法介绍一下

#### setSystemProcess

```java
public void setSystemProcess() {
    try {
        //将自己添加到ServiceManager中
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_HIGH);
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this),
                    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        }
        //将权限服务添加到ServiceManager中
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));
        ServiceManager.addService("cacheinfo", new CacheBinder(this));
        //查询包名为android的application，即framework-res.apk的application信息
        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
        //设置系统application信息
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

        synchronized (this) {
            //创建一个ProcessRecord对象记录系统application信息
            ProcessRecord app = mProcessList.newProcessRecordLocked(info, info.processName,
                    false,
                    0,
                    new HostingRecord("system"));
            app.setPersistent(true);
            app.pid = MY_PID;
            app.getWindowProcessController().setPid(MY_PID);
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            //将系统application的ProcessRecord对象也添加进来，让AMS可以管理调度
            addPidLocked(app);
            mProcessList.updateLruProcessLocked(app, false, null);
            updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException(
                "Unable to find android system package", e);
    }

    // Start watching app ops after we and the package manager are up and running.
    mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
            new IAppOpsCallback.Stub() {
                @Override public void opChanged(int op, int uid, String packageName) {
                    if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                        if (getAppOpsManager().checkOpNoThrow(op, uid, packageName)
                                != AppOpsManager.MODE_ALLOWED) {
                            runInBackgroundDisabled(uid);
                        }
                    }
                }
            });

    final int[] cameraOp = {AppOpsManager.OP_CAMERA};
    mAppOpsService.startWatchingActive(cameraOp, new IAppOpsActiveCallback.Stub() {
        @Override
        public void opActiveChanged(int op, int uid, String packageName, boolean active) {
            cameraActiveChanged(uid, active);
        }
    });
}
```

这个方法主要将自己和一些其他系统服务注册到了`ServiceManager`中，然后通过`PMS`找到`framework-res.apk`的`application`信息，将它配置到系统`ActivityThread`中，然后根据这个`application`信息，创建出了一个记录系统`application`信息的`ProcessRecord`对象，并将其添加到`AMS`内部的列表中，这样后续`AMS`就可以管理调度系统`application`了

#### installSystemProviders

```java
public final void installSystemProviders() {
    List<ProviderInfo> providers;
    synchronized (this) {
        //这里找到的就是setSystemProcess中创建的framework-res.apk的application信息
        ProcessRecord app = mProcessList.mProcessNames.get("system", SYSTEM_UID);
        //找到所有和framework-res.apk相关的ContentProvider
        providers = generateApplicationProvidersLocked(app);
        if (providers != null) {
            for (int i=providers.size()-1; i>=0; i--) {
                ProviderInfo pi = (ProviderInfo)providers.get(i);
                if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                    Slog.w(TAG, "Not installing system proc provider " + pi.name
                            + ": not system .apk");
                    providers.remove(i);
                }
            }
        }
    }
    if (providers != null) {
        mSystemThread.installSystemProviders(providers);
    }

    synchronized (this) {
        mSystemProvidersInstalled = true;
    }
    mConstants.start(mContext.getContentResolver());
    mCoreSettingsObserver = new CoreSettingsObserver(this);
    mActivityTaskManager.installSystemProviders();
    mDevelopmentSettingsObserver = new DevelopmentSettingsObserver();
    SettingsToPropertiesMapper.start(mContext.getContentResolver());
    mOomAdjuster.initSettings();

    // Now that the settings provider is published we can consider sending
    // in a rescue party.
    RescueParty.onSettingsProviderPublished(mContext);
}
```