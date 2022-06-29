---
title: Android源码分析 - SystemServer（下）
date: 2022-01-29 16:36:00
tags: 
- Android源码
- SystemServer
categories: 
- [Android, 源码分析]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

上一篇文章[Android源码分析 - SystemServer（上）](https://juejin.cn/post/7054154169761923085)我们分析了`SystemServer`进程是怎么被启动起来的，今天这篇，我们来分析`SystemServer`进程启动后做了什么

# main

我们上一章中讲到，`Zygote`进程`fork`出子进程后，最终调用了`SystemServer.main`方法，`SystemServer`源代码在`frameworks/base/services/java/com/android/server/SystemServer.java`中，我们来看看做了什么

```java
public static void main(String[] args) {
    new SystemServer().run();
}
```

## 构造方法

非常简单，就是先new了一个`SystemServer`对象，然后调用它的`run`方法，我们先看一下构造方法

```java
public SystemServer() {
    //工厂模式
    mFactoryTestMode = FactoryTest.getMode();

    ... //记录启动信息

    //记录是否经历过重启
    mRuntimeRestart = "1".equals(SystemProperties.get("sys.boot_completed"));
}
```

### 工厂模式

首先，先从系统属性中获取工厂模式级别，有三种属性：

- `FACTORY_TEST_OFF`：正常模式
- `FACTORY_TEST_LOW_LEVEL`：低级别工厂模式，在此模式下，很多Service不会启动
- `FACTORY_TEST_HIGH_LEVEL`：高级别工厂模式，此模式与正常模式基本相同，略有区别

它们被定义在`frameworks/base/core/java/android/os/FactoryTest.java`中

## run

紧接着便开始执行`run`方法

```java
private void run() {
    ... //记录启动信息
    //如果没有设置时区，将时区设置为GMT
    String timezoneProperty = SystemProperties.get("persist.sys.timezone");
    if (timezoneProperty == null || timezoneProperty.isEmpty()) {
        Slog.w(TAG, "Timezone not set; setting to GMT.");
        SystemProperties.set("persist.sys.timezone", "GMT");
    }

    //设置区域与语言
    if (!SystemProperties.get("persist.sys.language").isEmpty()) {
        final String languageTag = Locale.getDefault().toLanguageTag();

        SystemProperties.set("persist.sys.locale", languageTag);
        SystemProperties.set("persist.sys.language", "");
        SystemProperties.set("persist.sys.country", "");
        SystemProperties.set("persist.sys.localevar", "");
    }

    //Binder事务发生阻塞时发出警告
    Binder.setWarnOnBlocking(true);
    //PackageManager相关
    PackageItemInfo.forceSafeLabels();
    ...
    //设置虚拟机库文件libart.so
    SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());
    //清除虚拟机内存增长上限，以获得更多内存
    VMRuntime.getRuntime().clearGrowthLimit();
    // Some devices rely on runtime fingerprint generation, so make sure
    // we've defined it before booting further.
    Build.ensureFingerprintProperty();
    //设置在访问环境变量前，需要明确指定用户
    Environment.setUserRequired(true);
    //设置标记，当发生BadParcelableException异常时保守处理，不要抛出异常
    BaseBundle.setShouldDefuse(true);
    //设置异常跟踪
    Parcel.setStackTraceParceling(true);
    //确保Binder调用优先级总为前台优先级
    BinderInternal.disableBackgroundScheduling(true);
    //设置Binder线程池最大数量
    BinderInternal.setMaxThreads(sMaxBinderThreads);
    //设置进程优先级为前台进程
    // Prepare the main looper thread (this thread).
    android.os.Process.setThreadPriority(
            android.os.Process.THREAD_PRIORITY_FOREGROUND);
    android.os.Process.setCanSelfBackground(false);
    //以当前线程作为MainLooper准备
    Looper.prepareMainLooper();
    Looper.getMainLooper().setSlowLogThresholdMs(
            SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);

    SystemServiceRegistry.sEnableServiceNotFoundWtf = true;

    //加载android_servers.so库
    System.loadLibrary("android_servers");
    //标记该进程的堆可分析
    initZygoteChildHeapProfiling();
    //Debug选项 - 开启一个线程用来监测FD泄漏
    if (Build.IS_DEBUGGABLE) {
        spawnFdLeakCheckThread();
    }
    //检查上次关机过程中是否失败
    performPendingShutdown();
    //初始化System Context
    createSystemContext();
    //创建并设置一些每个进程启动时都需要的一些模块 (TelephonyServiceManager, StatsServiceManager)
    ActivityThread.initializeMainlineModules();
    
    //创建SystemServiceManager（管理所有的系统Service）
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    mSystemServiceManager.setStartInfo(mRuntimeRestart,
            mRuntimeStartElapsedTime, mRuntimeStartUptime);
    //将SystemServiceManager作为本地进程Service使用
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
    //为初始化任务准备线程池
    SystemServerInitThreadPool.start();
    ...
    //设置默认异常处理程序
    RuntimeInit.setDefaultApplicationWtfHandler(SystemServer::handleEarlySystemWtf);

    ...
    //启动引导服务
    startBootstrapServices(t);
    //启动核心服务
    startCoreServices(t);
    //启动其他服务
    startOtherServices(t);
    ...

    //严格模式初始化虚拟机策略
    StrictMode.initVmDefaults(null);
    ...
    //进入Looper死循环，等待Handler事件
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

可以看到，`run`方法主要做了以下工作

1. 检查并设置各种参数handler
2. 创建`SystemContext`
3. 创建`SystemServiceManager`
4. 启动服务
5. `Looper`循环

其中，创建`SystemContext`这一步是由`ContextImpl`完成的，等后面分析到的时候在详细去看，`Looper`也是，我们将重点放在启动服务上

# 启动服务

启动服务分为三步，首先是启动引导服务，其次是启动核心服务，最后是启动其他服务，我们先从引导服务开始

**由于启动的服务太多了，我们只介绍一些我们比较熟悉的服务**

## startBootstrapServices

```java
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
    ...
    //看门狗
    final Watchdog watchdog = Watchdog.getInstance();
    watchdog.start();
    ...
    final String TAG_SYSTEM_CONFIG = "ReadingSystemConfig";
    //读取系统配置
    SystemServerInitThreadPool.submit(SystemConfig::getInstance, TAG_SYSTEM_CONFIG);
    ...
    //Installer服务（实际上是与installd跨进程通信）
    Installer installer = mSystemServiceManager.startService(Installer.class);
    ...
    //创建 ATMS & AMS
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    mWindowManagerGlobalLock = atm.getGlobalLock();
    ...
    //电源管理服务，后面有其他服务依赖它，所以需要较早启动
    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
    ...
    mActivityManagerService.initPowerManagement();
    ...
    //灯光服务
    mSystemServiceManager.startService(LightsService.class);
    ...
    //显示管理服务
    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
    ...
    //阶段100
    mSystemServiceManager.startBootPhase(t, SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
    
    ...
    //创建PMS
    try {
        Watchdog.getInstance().pauseWatchingCurrentThread("packagemanagermain");
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
    } finally {
        Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
    }

    //捕获dex load行为
    SystemServerDexLoadReporter.configureSystemServerDexReporter(mPackageManagerService);
    //是否首次启动
    mFirstBoot = mPackageManagerService.isFirstBoot();
    //获取PMS
    mPackageManager = mSystemContext.getPackageManager();
    ...
    //用户管理服务
    mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
    ...
    //初始化属性缓存
    AttributeCache.init(mSystemContext);
    ...
    //注册各种系统服务
    mActivityManagerService.setSystemProcess();
    ...
    //使用AMS完成看门狗的设置，并监听重新启动
    watchdog.init(mSystemContext, mActivityManagerService);
    ...
    //设置调度策略
    mDisplayManagerService.setupSchedulerPolicies();
    ...
    //在单独线程中启动传感器服务
    mSensorServiceStart = SystemServerInitThreadPool.submit(() -> {
        TimingsTraceAndSlog traceLog = TimingsTraceAndSlog.newAsyncLog();
        traceLog.traceBegin(START_SENSOR_SERVICE);
        startSensorService();
        traceLog.traceEnd();
    }, START_SENSOR_SERVICE);
    ...
}
```

## startCoreServices

```java
private void startCoreServices(@NonNull TimingsTraceAndSlog t) {
    ...
    //电池电量服务，依赖LightsService
    mSystemServiceManager.startService(BatteryService.class);
    ...
    //应用统计服务
    mSystemServiceManager.startService(UsageStatsService.class);
    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));
    ...
}
```

## startOtherServices

```java
private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
    ...
    //AccountManagerService - 账户管理
    mSystemServiceManager.startService(ACCOUNT_SERVICE_CLASS);
    ...
    //ContentService - 内容服务
    mSystemServiceManager.startService(CONTENT_SERVICE_CLASS);
    ...
    //加载SettingProvider
    mActivityManagerService.installSystemProviders();
    ...
    //DropBox日志服务
    mSystemServiceManager.startService(DropBoxManagerService.class);
    ...
    //震动服务
    vibrator = new VibratorService(context);
    ServiceManager.addService("vibrator", vibrator);
    ...
    //时钟/闹钟服务
    mSystemServiceManager.startService(new AlarmManagerService(context));
    //输入服务
    inputManager = new InputManagerService(context);
    ...
    //等待传感器服务准备完毕
    ConcurrentUtils.waitForFutureNoInterrupt(mSensorServiceStart, START_SENSOR_SERVICE);
    mSensorServiceStart = null;
    //启动WindowManagerService
    wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
            new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
    ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
            DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
    ServiceManager.addService(Context.INPUT_SERVICE, inputManager,
            /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
    ...
    mActivityManagerService.setWindowManager(wm);
    ...
    wm.onInitReady();
    ...
    //HIDL services
    SystemServerInitThreadPool.submit(() -> {
        startHidlServices();
    }, START_HIDL_SERVICES);
    ...
    //关联WMS，启动输入服务
    inputManager.setWindowManagerCallbacks(wm.getInputManagerCallback());
    inputManager.start();
    ...
    mDisplayManagerService.windowManagerAndInputReady();
    ...
    //有蓝牙功能且非低级工厂模式，启动蓝牙服务
    if (mFactoryTestMode == FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        ...
    } else if (!context.getPackageManager().hasSystemFeature
            (PackageManager.FEATURE_BLUETOOTH)) {
        ...
    } else {
        mSystemServiceManager.startService(BluetoothService.class);
    }
    ...
    //输入法/无障碍服务
    if (mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        if (InputMethodSystemProperty.MULTI_CLIENT_IME_ENABLED) {
            mSystemServiceManager.startService(
                    MultiClientInputMethodManagerService.Lifecycle.class);
        } else {
            mSystemServiceManager.startService(InputMethodManagerService.Lifecycle.class);
        }
        mSystemServiceManager.startService(ACCESSIBILITY_MANAGER_SERVICE_CLASS);
    }

    wm.displayReady();
    
    //存储相关服务
    if (mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        if (!"0".equals(SystemProperties.get("system_init.startmountservice"))) {
            mSystemServiceManager.startService(STORAGE_MANAGER_SERVICE_CLASS);
            storageManager = IStorageManager.Stub.asInterface(
                        ServiceManager.getService("mount"));
            mSystemServiceManager.startService(STORAGE_STATS_SERVICE_CLASS);
        }
    }

    //UIMode服务（夜间模式，驾驶模式等）
    mSystemServiceManager.startService(UiModeManagerService.class);
    ...
    //执行磁盘清理工作，释放磁盘空间
    mPackageManagerService.performFstrimIfNeeded();

    if (mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        ...
        final boolean hasPdb = !SystemProperties.get(PERSISTENT_DATA_BLOCK_PROP).equals("");
        ...
        if (hasPdb || OemLockService.isHalPresent()) {
            //OEM锁服务
            mSystemServiceManager.startService(OemLockService.class);
        }
        ...
        if (!isWatch) {
            //状态栏管理服务
            statusBar = new StatusBarManagerService(context);
            ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar);
        }
        
        //网络相关服务
        ConnectivityModuleConnector.getInstance().init(context);
        NetworkStackClient.getInstance().init();
        networkManagement = NetworkManagementService.create(context);
        ServiceManager.addService(Context.NETWORKMANAGEMENT_SERVICE, networkManagement);
        ipSecService = IpSecService.create(context, networkManagement);
        ServiceManager.addService(Context.IPSEC_SERVICE, ipSecService);
        
        //文本服务
        mSystemServiceManager.startService(TextServicesManagerService.Lifecycle.class);
        mSystemServiceManager
                    .startService(TextClassificationManagerService.Lifecycle.class);

        //网络相关服务
        mSystemServiceManager.startService(NetworkScoreService.Lifecycle.class);
        networkStats = NetworkStatsService.create(context, networkManagement);
        ServiceManager.addService(Context.NETWORK_STATS_SERVICE, networkStats);
        networkPolicy = new NetworkPolicyManagerService(context, mActivityManagerService,
                    networkManagement);
        ServiceManager.addService(Context.NETWORK_POLICY_SERVICE, networkPolicy);
        if (context.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_WIFI)) {
            mSystemServiceManager.startServiceFromJar(
                    WIFI_SERVICE_CLASS, WIFI_APEX_SERVICE_JAR_PATH);
            mSystemServiceManager.startServiceFromJar(
                    WIFI_SCANNING_SERVICE_CLASS, WIFI_APEX_SERVICE_JAR_PATH);
        }
        if (context.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_WIFI_RTT)) {
            mSystemServiceManager.startServiceFromJar(
                    WIFI_RTT_SERVICE_CLASS, WIFI_APEX_SERVICE_JAR_PATH);
        }
        if (context.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_WIFI_AWARE)) {
            mSystemServiceManager.startServiceFromJar(
                    WIFI_AWARE_SERVICE_CLASS, WIFI_APEX_SERVICE_JAR_PATH);
        }
        if (context.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_WIFI_DIRECT)) {
            mSystemServiceManager.startServiceFromJar(
                    WIFI_P2P_SERVICE_CLASS, WIFI_APEX_SERVICE_JAR_PATH);
        }
        if (context.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_LOWPAN)) {
            mSystemServiceManager.startService(LOWPAN_SERVICE_CLASS);
        }
        if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_ETHERNET) ||
                mPackageManager.hasSystemFeature(PackageManager.FEATURE_USB_HOST)) {
            mSystemServiceManager.startService(ETHERNET_SERVICE_CLASS);
        }
        connectivity = new ConnectivityService(
                    context, networkManagement, networkStats, networkPolicy);
        ServiceManager.addService(Context.CONNECTIVITY_SERVICE, connectivity,
                    /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_HIGH | DUMP_FLAG_PRIORITY_NORMAL);
        networkPolicy.bindConnectivityManager(connectivity);
        ...
        //系统更新服务
        ServiceManager.addService(Context.SYSTEM_UPDATE_SERVICE,
                    new SystemUpdateManagerService(context));
        ServiceManager.addService(Context.UPDATE_LOCK_SERVICE,
                    new UpdateLockService(context));
        //通知服务
        mSystemServiceManager.startService(NotificationManagerService.class);
        SystemNotificationChannels.removeDeprecated(context);
        SystemNotificationChannels.createAll(context);
        notification = INotificationManager.Stub.asInterface(
                ServiceManager.getService(Context.NOTIFICATION_SERVICE));
        ...
        //位置服务
        mSystemServiceManager.startService(LocationManagerService.Lifecycle.class);
        ...
        //墙纸服务
        if (context.getResources().getBoolean(R.bool.config_enableWallpaperService)) {
            mSystemServiceManager.startService(WALLPAPER_SERVICE_CLASS);
        } else {
            ...
        }
        //音频服务
        if (!isArc) {
            mSystemServiceManager.startService(AudioService.Lifecycle.class);
        } else {
            String className = context.getResources()
                    .getString(R.string.config_deviceSpecificAudioService);
            mSystemServiceManager.startService(className + "$Lifecycle");
        }
        ...
        //ADB服务
        mSystemServiceManager.startService(ADB_SERVICE_CLASS);
        //USB服务
        if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_USB_HOST)
                || mPackageManager.hasSystemFeature(
                PackageManager.FEATURE_USB_ACCESSORY)
                || isEmulator) {
            mSystemServiceManager.startService(USB_SERVICE_CLASS);
        }
        //微件（小组件）服务
        if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_APP_WIDGETS)
                || context.getResources().getBoolean(R.bool.config_enableAppWidgetService)) {
            mSystemServiceManager.startService(APPWIDGET_SERVICE_CLASS);
        }
        ...
        //Android10新增，用于报告来自运行时模块的信息
        ServiceManager.addService("runtime", new RuntimeService(context));
        ...
        //App后台Dex优化
        BackgroundDexOptService.schedule(context);
        ...
    }
    ...
    //相机服务
    if (!disableCameraService) {
        mSystemServiceManager.startService(CameraServiceProxy.class);
    }
    //进入安全模式
    if (safeMode) {
        mActivityManagerService.enterSafeMode();
    }

    //短信服务
    mmsService = mSystemServiceManager.startService(MmsServiceBroker.class);
    ...
    //剪贴板服务
    mSystemServiceManager.startService(ClipboardService.class);
    ...
    
    //调用各大服务的systemReady方法

    vibrator.systemReady();
    lockSettings.systemReady();
    
    //阶段480
    mSystemServiceManager.startBootPhase(t, SystemService.PHASE_LOCK_SETTINGS_READY);
    //阶段500
    mSystemServiceManager.startBootPhase(t, SystemService.PHASE_SYSTEM_SERVICES_READY);

    wm.systemReady();
    ...
    //手动更新Context Configuration
    final Configuration config = wm.computeNewConfiguration(DEFAULT_DISPLAY);
    DisplayMetrics metrics = new DisplayMetrics();
    context.getDisplay().getMetrics(metrics);
    context.getResources().updateConfiguration(config, metrics);

    final Theme systemTheme = context.getTheme();
    if (systemTheme.getChangingConfigurations() != 0) {
        systemTheme.rebase();
    }

    mPowerManagerService.systemReady(mActivityManagerService.getAppOpsService());
    ...
    mPackageManagerService.systemReady();
    mDisplayManagerService.systemReady(safeMode, mOnlyCore);
    
    mSystemServiceManager.setSafeMode(safeMode);

    //阶段520
    mSystemServiceManager.startBootPhase(t, SystemService.PHASE_DEVICE_SPECIFIC_SERVICES_READY);
    ...
    //最后运行AMS.systemReady
    mActivityManagerService.systemReady(() -> {
        //阶段550
        mSystemServiceManager.startBootPhase(t, SystemService.PHASE_ACTIVITY_MANAGER_READY);
        ...
        //阶段600
        mSystemServiceManager.startBootPhase(t, SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
        ...
    }, t);
}
```

服务的启动是分阶段完成的，从0-100-480-500-520-550-600-1000，最后的阶段1000，是在AMS调用`finishBooting`方法后进入

可以看到，启动的服务非常之多，不可能全看得完，其中最重要的几个：`ActivityManagerService`、`WindowManagerService`、`PackageManagerService`和`InputManagerService`，后面我们会慢慢看过去，在此之前，我们还是先看看服务启动的方式

# SystemServiceManager

绝大部分的服务是通过`SystemServiceManager`启动的，它的源码路径为`frameworks/base/services/core/java/com/android/server/SystemServiceManager.java`

## startService

我们来看看这个类里的启动服务方法

这个类中有三个方法用于启动Serivce，分别是：

- `public SystemService startService(String className)`
- `public SystemService startServiceFromJar(String className, String path)`
- `public <T extends SystemService> T startService(Class<T> serviceClass)`
- `public void startService(@NonNull final SystemService service)`

实际上最后都是调用了最后一个方法

先看参数为`String`的`startService`方法

```java
public SystemService startService(String className) {
    final Class<SystemService> serviceClass = loadClassFromLoader(className,
            this.getClass().getClassLoader());
    return startService(serviceClass);
}
```

```java
private static Class<SystemService> loadClassFromLoader(String className,
        ClassLoader classLoader) {
    try {
        return (Class<SystemService>) Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        ...
    }
}
```

实际上就是通过反射拿到类名对应的`Class`，再调用`Class`为参的`startService`方法

`startServiceFromJar`实际上也是一样，只不过是先通过`PathClassLoader`加载了jar而已

```java
public SystemService startServiceFromJar(String className, String path) {
    PathClassLoader pathClassLoader = mLoadedPaths.get(path);
    if (pathClassLoader == null) {
        // NB: the parent class loader should always be the system server class loader.
        // Changing it has implications that require discussion with the mainline team.
        pathClassLoader = new PathClassLoader(path, this.getClass().getClassLoader());
        mLoadedPaths.put(path, pathClassLoader);
    }
    final Class<SystemService> serviceClass = loadClassFromLoader(className, pathClassLoader);
    return startService(serviceClass);
}
```

接着我们看看`Class`为参数的`startService`方法

```java
public <T extends SystemService> T startService(Class<T> serviceClass) {
    final String name = serviceClass.getName();

    // Create the service.
    if (!SystemService.class.isAssignableFrom(serviceClass)) {
        throw new RuntimeException("Failed to create " + name
                + ": service must extend " + SystemService.class.getName());
    }
    final T service;
    try {
        Constructor<T> constructor = serviceClass.getConstructor(Context.class);
        service = constructor.newInstance(mContext);
    } catch (...) {
        ...
    }

    startService(service);
    return service;
}
```

看函数泛型我们就可以知道，这个方法只接受`SystemService`的子类，并且在方法的开头，还使用了`isAssignableFrom`方法做了类型校验，避免通过`String`反射获取的`Class`非`SystemService`的子类

之后的逻辑也很简单，反射实例化对象，然后调用另一个以`SystemService`对象为参数的重载方法

```java
public void startService(@NonNull final SystemService service) {
    mServices.add(service);
    try {
        service.onStart();
    } catch (RuntimeException ex) {
        throw new RuntimeException("Failed to start service " + service.getClass().getName()
                + ": onStart threw an exception", ex);
    }
}
```

这个方法会将`SystemService`对象加入一个`List`中，然后调用它的`onStart`方法，通知`SystemService`自行处理启动

## startBootPhase

因为各种服务之间是存在依赖关系的，所以Android将服务的启动划分了8个阶段：0-100-480-500-520-550-600-1000，而`startBootPhase`方法便是用来通知各个服务进行到哪一阶段了

```java
public void startBootPhase(@NonNull TimingsTraceAndSlog t, int phase) {
    if (phase <= mCurrentPhase) {
        throw new IllegalArgumentException("Next phase must be larger than previous");
    }
    mCurrentPhase = phase;

    final int serviceLen = mServices.size();
    for (int i = 0; i < serviceLen; i++) {
        final SystemService service = mServices.get(i);
        service.onBootPhase(mCurrentPhase);
    }

    if (phase == SystemService.PHASE_BOOT_COMPLETED) {
        SystemServerInitThreadPool.shutdown();
    }
}
```

每进入到一个阶段，便会调用Service List中所有`SystemService`的`onBootPhase`方法，通知`SystemService`阶段变换，而当阶段达到1000 (PHASE_BOOT_COMPLETED) 时，就代表着所有的服务都已准备完毕，关闭`SystemServerInitThreadPool`线程池

# ServiceManager

当服务被创建出来后，会调用`ServiceManager.addService`方法添加服务，以供其他地方使用这些服务

`addService`有三个重载，最终调用的为：

```java
public static void addService(String name, IBinder service, boolean allowIsolated,
        int dumpPriority) {
    try {
        getIServiceManager().addService(name, service, allowIsolated, dumpPriority);
    } catch (RemoteException e) {
        Log.e(TAG, "error in addService", e);
    }
}
```

```java
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }

    // Find the service manager
    sServiceManager = ServiceManagerNative
            .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
    return sServiceManager;
}
```

```java
public static IServiceManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }

    // ServiceManager is never local
    return new ServiceManagerProxy(obj);
}
```

```java
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
        mServiceManager = IServiceManager.Stub.asInterface(remote);
    }

    public IBinder asBinder() {
        return mRemote;
    }
    ...
    public void addService(String name, IBinder service, boolean allowIsolated, int dumpPriority)
            throws RemoteException {
        mServiceManager.addService(name, service, allowIsolated, dumpPriority);
    }
    ...
    private IBinder mRemote;

    private IServiceManager mServiceManager;
}
```

从这里就能看出来`ServiceManager`实际上是一个单独的进程，名为`servicemanager`，它负责管理所有服务，使用了`Binder` IPC机制，我们调用`addService`方法实际上是调用了Binder Proxy的方法，他向`/dev/binder`中写入消息，在`servicemanager`进程中接收到了这个消息并处理这个请求

关于`Binder`机制，我们随后便会分析它

最终调用了`frameworks/native/cmds/servicemanager/ServiceManager.cpp`中的`addService`函数

```c++
Status ServiceManager::addService(const std::string& name, const sp<IBinder>& binder, bool allowIsolated, int32_t dumpPriority) {
    auto ctx = mAccess->getCallingContext();
    ...
    //添加服务
    auto entry = mNameToService.emplace(name, Service {
        .binder = binder,
        .allowIsolated = allowIsolated,
        .dumpPriority = dumpPriority,
        .debugPid = ctx.debugPid,
    });

    auto it = mNameToRegistrationCallback.find(name);
    if (it != mNameToRegistrationCallback.end()) {
        for (const sp<IServiceCallback>& cb : it->second) {
            entry.first->second.guaranteeClient = true;
            // permission checked in registerForNotifications
            cb->onRegistration(name, binder);
        }
    }

    return Status::ok();
}
```

可以看到，最终通过`service name`和传过来的`binder`对象构造出一个`Service`结构体，并将其保存至`mNameToService`这个Map中，以供后面使用

# 关于进程

`SystemServer`启动的服务大多都运行在`systemserver`进程中，但也有一些例外

譬如`Installer`服务，便是从`init`进程单独`fork`出了一个`installd`进程

下面是它的rc文件，`frameworks/native/cmds/installd/installd.rc`

```
service installd /system/bin/installd
class main
...
```

而在`SystemServer`进程中start的`Installer`，便是通过`binder`连接到`installd`进程提供服务

源码路径`frameworks/base/services/core/java/com/android/server/pm/Installer.java`

```java
@Override
public void onStart() {
    if (mIsolated) {
        mInstalld = null;
    } else {
        connect();
    }
}

private void connect() {
    IBinder binder = ServiceManager.getService("installd");
    if (binder != null) {
        try {
            binder.linkToDeath(new DeathRecipient() {
                @Override
                public void binderDied() {
                    Slog.w(TAG, "installd died; reconnecting");
                    connect();
                }
            }, 0);
        } catch (RemoteException e) {
            binder = null;
        }
    }

    if (binder != null) {
        mInstalld = IInstalld.Stub.asInterface(binder);
        try {
            invalidateMounts();
        } catch (InstallerException ignored) {
        }
    } else {
        Slog.w(TAG, "installd not found; trying again");
        BackgroundThread.getHandler().postDelayed(() -> {
            connect();
        }, DateUtils.SECOND_IN_MILLIS);
    }
}
```

# 结束

`SystemServer`启动了非常多的服务，并将这些服务添加到了`ServiceManager`中，我们又从中引申出了`Binder`机制，我们下一章便开始分析`Binder`