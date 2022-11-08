---
title: Android源码分析 - Activity启动流程（下）
date: 2022-10-24 14:49:22
tags: 
- Android源码
- ActivityThread
- ActivityManagerService
categories: 
- [Android, 源码分析]
- [Android, ActivityThread]
- [Android, ActivityManagerService]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

上一篇文章 [Android源码分析 - Activity启动流程（上）](https://juejin.cn/post/7130182223231188999) 中，我们分析了`Activity`启动流程中的一小部分，基本上可以算是`Activity`启动的前置准备工作，这篇文章我们将会分析`Activity`启动的主要流程

# 启动App进程

我们从App进程的启动开始说起，上篇文章中我们说过了，如果App尚未启动，则会调用`ATMS`的`startProcessAsync`方法去启动App进程

```java
void startProcessAsync(ActivityRecord activity, boolean knownToBeDead, boolean isTop,
        String hostingType) {
    try {
        ...
        // Post message to start process to avoid possible deadlock of calling into AMS with the
        // ATMS lock held.
        final Message m = PooledLambda.obtainMessage(ActivityManagerInternal::startProcess,
                mAmInternal, activity.processName, activity.info.applicationInfo, knownToBeDead,
                isTop, hostingType, activity.intent.getComponent());
        mH.sendMessage(m);
    } finally {
        ...
    }
}
```

这个方法实际上是通过`Hander`调用了`ActivityManagerInternal (AMS.LocalService)`的`startProcess`方法

```java
@Override
public void startProcess(String processName, ApplicationInfo info, boolean knownToBeDead,
    boolean isTop, String hostingType, ComponentName hostingName) {
try {
    ...
    synchronized (ActivityManagerService.this) {
        // If the process is known as top app, set a hint so when the process is
        // started, the top priority can be applied immediately to avoid cpu being
        // preempted by other processes before attaching the process of top app.
        startProcessLocked(processName, info, knownToBeDead, 0 /* intentFlags */,
                new HostingRecord(hostingType, hostingName, isTop),
                ZYGOTE_POLICY_FLAG_LATENCY_SENSITIVE, false /* allowWhileBooting */,
                false /* isolated */, true /* keepIfLarge */);
    }
} finally {
    ...
}
}
```

这里将进程启动的一些信息封装到了`HostingRecord`类中

```java
final ProcessRecord startProcessLocked(String processName,
    ApplicationInfo info, boolean knownToBeDead, int intentFlags,
    HostingRecord hostingRecord, int zygotePolicyFlags, boolean allowWhileBooting,
    boolean isolated, boolean keepIfLarge) {
return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
        hostingRecord, zygotePolicyFlags, allowWhileBooting, isolated, 0 /* isolatedUid */,
        keepIfLarge, null /* ABI override */, null /* entryPoint */,
        null /* entryPointArgs */, null /* crashHandler */);
}
```

`AMS`将启动进程的任务转交给了`ProcessList`，这个类的职责是管理进程，包括管理进程优先级(Adj)、进程OOM等

```java
final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
        boolean knownToBeDead, int intentFlags, HostingRecord hostingRecord,
        int zygotePolicyFlags, boolean allowWhileBooting, boolean isolated, int isolatedUid,
        boolean keepIfLarge, String abiOverride, String entryPoint, String[] entryPointArgs,
        Runnable crashHandler) {
    long startTime = SystemClock.uptimeMillis();
    ProcessRecord app;
    if (!isolated) {
        //先通过进程名和uid查找相应App的ProcessRecord
        app = getProcessRecordLocked(processName, info.uid, keepIfLarge);

        //如果是由后台进程发起的 startProcess
        //判断启动进程是否为 bad process，如果是，直接启动失败返回
        //这里 bad process 的定义为：短时间内连续崩溃两次以上的进程
        if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
            // If we are in the background, then check to see if this process
            // is bad.  If so, we will just silently fail.
            if (mService.mAppErrors.isBadProcessLocked(info)) {
                return null;
            }
        } else {
            // When the user is explicitly starting a process, then clear its
            // crash count so that we won't make it bad until they see at
            // least one crash dialog again, and make the process good again
            // if it had been bad.
            //如果是用户显式的要求启动进程，则会清空启动进程的崩溃次数，将启动进程从 bad process 列表中移除
            mService.mAppErrors.resetProcessCrashTimeLocked(info);
            if (mService.mAppErrors.isBadProcessLocked(info)) {
                EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                        UserHandle.getUserId(info.uid), info.uid,
                        info.processName);
                mService.mAppErrors.clearBadProcessLocked(info);
                if (app != null) {
                    app.bad = false;
                }
            }
        }
    } else {
        // If this is an isolated process, it can't re-use an existing process.
        app = null;
    }

    // We don't have to do anything more if:
    // (1) There is an existing application record; and
    // (2) The caller doesn't think it is dead, OR there is no thread
    //     object attached to it so we know it couldn't have crashed; and
    // (3) There is a pid assigned to it, so it is either starting or
    //     already running.
    ProcessRecord precedence = null;
    //如果已经存在了对应App的ProcessRecord，并且分配了pid
    if (app != null && app.pid > 0) {
        //如果进程没有死亡或者进程还未绑定binder线程，说明进程是正常运行状态或正在启动中
        if ((!knownToBeDead && !app.killed) || app.thread == null) {
            // We already have the app running, or are waiting for it to
            // come up (we have a pid but not yet its thread), so keep it.
            // If this is a new package in the process, add the package to the list
            //将要启动的包信息记录在ProcessRecord中（Android多个App可以运行在同一个进程中）
            app.addPackage(info.packageName, info.longVersionCode, mService.mProcessStats);
            return app;
        }

        // An application record is attached to a previous process,
        // clean it up now.
        //App绑定在之前的一个进程上了，杀死并清理这个进程
        ProcessList.killProcessGroup(app.uid, app.pid);

        Slog.wtf(TAG_PROCESSES, app.toString() + " is attached to a previous process");
        // We are not going to re-use the ProcessRecord, as we haven't dealt with the cleanup
        // routine of it yet, but we'd set it as the precedence of the new process.
        precedence = app;
        app = null;
    }

    //没有找到对应的ProcessRecord
    if (app == null) {
        //新创建一个ProcessRecord对象
        app = newProcessRecordLocked(info, processName, isolated, isolatedUid, hostingRecord);
        if (app == null) {
            Slog.w(TAG, "Failed making new process record for "
                    + processName + "/" + info.uid + " isolated=" + isolated);
            return null;
        }
        app.crashHandler = crashHandler;
        app.isolatedEntryPoint = entryPoint;
        app.isolatedEntryPointArgs = entryPointArgs;
        if (precedence != null) {
            app.mPrecedence = precedence;
            precedence.mSuccessor = app;
        }
    } else {    //存在对应的ProcessRecord，但进程尚未启动或已被清理
        // If this is a new package in the process, add the package to the list
        //将要启动的包信息记录在ProcessRecord中
        app.addPackage(info.packageName, info.longVersionCode, mService.mProcessStats);
    }

    // If the system is not ready yet, then hold off on starting this
    // process until it is.
    //如果系统尚未准备好（开机中或system_server进程崩溃重启中），将其先添加到等待队列中
    if (!mService.mProcessesReady
            && !mService.isAllowedWhileBooting(info)
            && !allowWhileBooting) {
        if (!mService.mProcessesOnHold.contains(app)) {
            mService.mProcessesOnHold.add(app);
        }
        return app;
    }

    final boolean success =
            startProcessLocked(app, hostingRecord, zygotePolicyFlags, abiOverride);
    return success ? app : null;
}
```

这个方法主要是处理`ProcessRecord`对象，如果找不到对应的`ProcessRecord`或对应的`ProcessRecord`里的信息表明App进程尚未启动，则会调用另一个`startProcessLocked`重载方法启动进程

```java
final boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
        int zygotePolicyFlags, String abiOverride) {
    return startProcessLocked(app, hostingRecord, zygotePolicyFlags,
            false /* disableHiddenApiChecks */, false /* disableTestApiChecks */,
            false /* mountExtStorageFull */, abiOverride);
}

boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
        int zygotePolicyFlags, boolean disableHiddenApiChecks, boolean disableTestApiChecks,
        boolean mountExtStorageFull, String abiOverride) {
    //进程正在启动中
    if (app.pendingStart) {
        return true;
    }
    //从刚才方法中的判断来看，应该不会进入这个case
    if (app.pid > 0 && app.pid != ActivityManagerService.MY_PID) {
        //将ProcessRecord的pid从PidMap中移除
        mService.removePidLocked(app);
        app.bindMountPending = false;
        //将ProcessRecord的pid重置为0
        app.setPid(0);
        app.startSeq = 0;
    }

    //将ProcessRecord从启动等待队列中移除
    mService.mProcessesOnHold.remove(app);

    mService.updateCpuStats();

    try {
        try {
            //检测当前用户是否可以启动这个App
            final int userId = UserHandle.getUserId(app.uid);
            AppGlobals.getPackageManager().checkPackageStartable(app.info.packageName, userId);
        } catch (RemoteException e) {
            throw e.rethrowAsRuntimeException();
        }

        int uid = app.uid;
        int[] gids = null;
        //默认不挂载外置存储
        int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
        if (!app.isolated) {
            int[] permGids = null;
            try {
                final IPackageManager pm = AppGlobals.getPackageManager();
                //获取GIDS（App申请的权限）
                permGids = pm.getPackageGids(app.info.packageName,
                        MATCH_DIRECT_BOOT_AUTO, app.userId);
                if (StorageManager.hasIsolatedStorage() && mountExtStorageFull) {
                    //挂载外置存储，允许读写
                    mountExternal = Zygote.MOUNT_EXTERNAL_FULL;
                } else {
                    StorageManagerInternal storageManagerInternal = LocalServices.getService(
                            StorageManagerInternal.class);
                    //获取App对外置存储的读写权限
                    mountExternal = storageManagerInternal.getExternalStorageMountMode(uid,
                            app.info.packageName);
                }
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }

            // Remove any gids needed if the process has been denied permissions.
            // NOTE: eventually we should probably have the package manager pre-compute
            // this for us?
            //从刚刚过去的App申请权限中剔除进程所被拒绝的权限
            if (app.processInfo != null && app.processInfo.deniedPermissions != null) {
                for (int i = app.processInfo.deniedPermissions.size() - 1; i >= 0; i--) {
                    int[] denyGids = mService.mPackageManagerInt.getPermissionGids(
                            app.processInfo.deniedPermissions.valueAt(i), app.userId);
                    if (denyGids != null) {
                        for (int gid : denyGids) {
                            permGids = ArrayUtils.removeInt(permGids, gid);
                        }
                    }
                }
            }

            //计算得出进程所应拥有的所有权限
            gids = computeGidsForProcess(mountExternal, uid, permGids);
        }
        //设置挂载模式
        app.mountMode = mountExternal;
        //工厂测试进程
        if (mService.mAtmInternal.isFactoryTestProcess(app.getWindowProcessController())) {
            uid = 0;
        }
        //进程启动参数（传递到Zygoto）
        int runtimeFlags = 0;
        //如果manifest中设置了android:debuggable
        if ((app.info.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0) {
            runtimeFlags |= Zygote.DEBUG_ENABLE_JDWP;
            runtimeFlags |= Zygote.DEBUG_JAVA_DEBUGGABLE;
            // Also turn on CheckJNI for debuggable apps. It's quite
            // awkward to turn on otherwise.
            runtimeFlags |= Zygote.DEBUG_ENABLE_CHECKJNI;

            // Check if the developer does not want ART verification
            if (android.provider.Settings.Global.getInt(mService.mContext.getContentResolver(),
                    android.provider.Settings.Global.ART_VERIFIER_VERIFY_DEBUGGABLE, 1) == 0) {
                runtimeFlags |= Zygote.DISABLE_VERIFIER;
                Slog.w(TAG_PROCESSES, app + ": ART verification disabled");
            }
        }
        ... //设置各种高进程启动参数

        String invokeWith = null;
        //如果manifest中设置了android:debuggable
        //使用logwrapper工具捕获stdout信息
        if ((app.info.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0) {
            // Debuggable apps may include a wrapper script with their library directory.
            String wrapperFileName = app.info.nativeLibraryDir + "/wrap.sh";
            StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskReads();
            try {
                if (new File(wrapperFileName).exists()) {
                    invokeWith = "/system/bin/logwrapper " + wrapperFileName;
                }
            } finally {
                StrictMode.setThreadPolicy(oldPolicy);
            }
        }

        //确定App进程使用的abi（有so库的App会通过so库的架构决定，没有so库的使用系统最优先支持的abi）
        String requiredAbi = (abiOverride != null) ? abiOverride : app.info.primaryCpuAbi;
        if (requiredAbi == null) {
            requiredAbi = Build.SUPPORTED_ABIS[0];
        }

        //将abi转成InstructionSet
        String instructionSet = null;
        if (app.info.primaryCpuAbi != null) {
            instructionSet = VMRuntime.getInstructionSet(app.info.primaryCpuAbi);
        }

        app.gids = gids;
        app.setRequiredAbi(requiredAbi);
        app.instructionSet = instructionSet;

        ...

        final String seInfo = app.info.seInfo
                + (TextUtils.isEmpty(app.info.seInfoUser) ? "" : app.info.seInfoUser);
        // Start the process.  It will either succeed and return a result containing
        // the PID of the new process, or else throw a RuntimeException.
        //重要：设置进程启动入口
        final String entryPoint = "android.app.ActivityThread";

        //启动进程
        return startProcessLocked(hostingRecord, entryPoint, app, uid, gids,
                runtimeFlags, zygotePolicyFlags, mountExternal, seInfo, requiredAbi,
                instructionSet, invokeWith, startTime);
    } catch (RuntimeException e) {
        ...
        mService.forceStopPackageLocked(app.info.packageName, UserHandle.getAppId(app.uid),
                false, false, true, false, false, app.userId, "start failure");
        return false;
    }
}
```

到这一步位置仍然是在进行准备工作，主要做了以下几件事：

1. 权限处理：App安装时会检测`manifest`里申请的权限，并由此生成出一个`GIDS`数组
2. 设置挂载模式
3. 设置进程的各种启动参数
4. 设置App进程使用的`abi`
5. 设置进程启动入口
6. 继续调用重载方法启动进程

```java
boolean startProcessLocked(HostingRecord hostingRecord, String entryPoint, ProcessRecord app,
        int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags, int mountExternal,
        String seInfo, String requiredAbi, String instructionSet, String invokeWith,
        long startTime) {
    //初始化一些参数
    //标识App进程正在启动
    app.pendingStart = true;
    app.killedByAm = false;
    app.removed = false;
    app.killed = false;
    app.mDisabledCompatChanges = null;
    if (mPlatformCompat != null) {
        app.mDisabledCompatChanges = mPlatformCompat.getDisabledChanges(app.info);
    }
    final long startSeq = app.startSeq = ++mProcStartSeqCounter;
    app.setStartParams(uid, hostingRecord, seInfo, startTime);
    app.setUsingWrapper(invokeWith != null
            || Zygote.getWrapProperty(app.processName) != null);
    mPendingStarts.put(startSeq, app);

    if (mService.mConstants.FLAG_PROCESS_START_ASYNC) {    //异步启动进程
        if (DEBUG_PROCESSES) Slog.i(TAG_PROCESSES,
                "Posting procStart msg for " + app.toShortString());
        mService.mProcStartHandler.post(() -> handleProcessStart(
                app, entryPoint, gids, runtimeFlags, zygotePolicyFlags, mountExternal,
                requiredAbi, instructionSet, invokeWith, startSeq));
        return true;
    } else {    //同步启动进程
        try {
            final Process.ProcessStartResult startResult = startProcess(hostingRecord,
                    entryPoint, app,
                    uid, gids, runtimeFlags, zygotePolicyFlags, mountExternal, seInfo,
                    requiredAbi, instructionSet, invokeWith, startTime);
            handleProcessStartedLocked(app, startResult.pid, startResult.usingWrapper,
                    startSeq, false);
        } catch (RuntimeException e) {
            //出错，将pendingStart标志复位并强行停止进程
            app.pendingStart = false;
            mService.forceStopPackageLocked(app.info.packageName, UserHandle.getAppId(app.uid),
                    false, false, true, false, false, app.userId, "start failure");
        }
        return app.pid > 0;
    }
}
```

在异步模式下，程序会等待`ProcessRecord.mPrecedence`进程结束才会启动进程（这里对应着最开始的`startProcessLocked`方法中，已经存在了对应App的`ProcessRecord`，并且分配了`pid`，但是进程被标记为死亡这种情况）

最终都会进入到`startProcess`和`handleProcessStartedLocked`方法中来

```java
private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,
        ProcessRecord app, int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags,
        int mountExternal, String seInfo, String requiredAbi, String instructionSet,
        String invokeWith, long startTime) {
    try {
        final boolean isTopApp = hostingRecord.isTopApp();
        if (isTopApp) {
            // Use has-foreground-activities as a temporary hint so the current scheduling
            // group won't be lost when the process is attaching. The actual state will be
            // refreshed when computing oom-adj.
            app.setHasForegroundActivities(true);
        }

        //处理应用目录隔离机制
        Map<String, Pair<String, Long>> pkgDataInfoMap;
        Map<String, Pair<String, Long>> whitelistedAppDataInfoMap;
        boolean bindMountAppStorageDirs = false;
        boolean bindMountAppsData = mAppDataIsolationEnabled
                && (UserHandle.isApp(app.uid) || UserHandle.isIsolated(app.uid))
                && mPlatformCompat.isChangeEnabled(APP_DATA_DIRECTORY_ISOLATION, app.info);

        // Get all packages belongs to the same shared uid. sharedPackages is empty array
        // if it doesn't have shared uid.
        final PackageManagerInternal pmInt = mService.getPackageManagerInternalLocked();
        final String[] sharedPackages = pmInt.getSharedUserPackagesForPackage(
                app.info.packageName, app.userId);
        final String[] targetPackagesList = sharedPackages.length == 0
                ? new String[]{app.info.packageName} : sharedPackages;

        pkgDataInfoMap = getPackageAppDataInfoMap(pmInt, targetPackagesList, uid);
        if (pkgDataInfoMap == null) {
            // TODO(b/152760674): Handle inode == 0 case properly, now we just give it a
            // tmp free pass.
            bindMountAppsData = false;
        }

        // Remove all packages in pkgDataInfoMap from mAppDataIsolationWhitelistedApps, so
        // it won't be mounted twice.
        final Set<String> whitelistedApps = new ArraySet<>(mAppDataIsolationWhitelistedApps);
        for (String pkg : targetPackagesList) {
            whitelistedApps.remove(pkg);
        }

        whitelistedAppDataInfoMap = getPackageAppDataInfoMap(pmInt,
                whitelistedApps.toArray(new String[0]), uid);
        if (whitelistedAppDataInfoMap == null) {
            // TODO(b/152760674): Handle inode == 0 case properly, now we just give it a
            // tmp free pass.
            bindMountAppsData = false;
        }

        int userId = UserHandle.getUserId(uid);
        StorageManagerInternal storageManagerInternal = LocalServices.getService(
                StorageManagerInternal.class);
        if (needsStorageDataIsolation(storageManagerInternal, app)) {
            bindMountAppStorageDirs = true;
            if (pkgDataInfoMap == null ||
                    !storageManagerInternal.prepareStorageDirs(userId, pkgDataInfoMap.keySet(),
                    app.processName)) {
                // Cannot prepare Android/app and Android/obb directory or inode == 0,
                // so we won't mount it in zygote, but resume the mount after unlocking device.
                app.bindMountPending = true;
                bindMountAppStorageDirs = false;
            }
        }

        // If it's an isolated process, it should not even mount its own app data directories,
        // since it has no access to them anyway.
        if (app.isolated) {
            pkgDataInfoMap = null;
            whitelistedAppDataInfoMap = null;
        }

        final Process.ProcessStartResult startResult;
        if (hostingRecord.usesWebviewZygote()) {
            startResult = startWebView(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, null, app.info.packageName, app.mDisabledCompatChanges,
                    new String[]{PROC_START_SEQ_IDENT + app.startSeq});
        } else if (hostingRecord.usesAppZygote()) {
            final AppZygote appZygote = createAppZygoteForProcessIfNeeded(app);

            // We can't isolate app data and storage data as parent zygote already did that.
            startResult = appZygote.getProcess().start(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, null, app.info.packageName,
                    /*zygotePolicyFlags=*/ ZYGOTE_POLICY_FLAG_EMPTY, isTopApp,
                    app.mDisabledCompatChanges, pkgDataInfoMap, whitelistedAppDataInfoMap,
                    false, false,
                    new String[]{PROC_START_SEQ_IDENT + app.startSeq});
        } else {    //没有特别指定hostingZygote时，进入此case
            startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, invokeWith, app.info.packageName, zygotePolicyFlags,
                    isTopApp, app.mDisabledCompatChanges, pkgDataInfoMap,
                    whitelistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs,
                    new String[]{PROC_START_SEQ_IDENT + app.startSeq});
        }
        return startResult;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

从`Android 11`开始引入了应用目录隔离机制，使得应用仅可以发现和访问自己的储存目录，不可以访问其他应用的储存目录

这里处理完应用目录隔离机制后，调用了`Process.start`方法启动进程，最终走到`ZygoteProcess.startViaZygote`方法

```java
private Process.ProcessStartResult startViaZygote(@NonNull final String processClass,
                                                    @Nullable final String niceName,
                                                    final int uid, final int gid,
                                                    @Nullable final int[] gids,
                                                    int runtimeFlags, int mountExternal,
                                                    int targetSdkVersion,
                                                    @Nullable String seInfo,
                                                    @NonNull String abi,
                                                    @Nullable String instructionSet,
                                                    @Nullable String appDataDir,
                                                    @Nullable String invokeWith,
                                                    boolean startChildZygote,
                                                    @Nullable String packageName,
                                                    int zygotePolicyFlags,
                                                    boolean isTopApp,
                                                    @Nullable long[] disabledCompatChanges,
                                                    @Nullable Map<String, Pair<String, Long>>
                                                            pkgDataInfoMap,
                                                    @Nullable Map<String, Pair<String, Long>>
                                                            allowlistedDataInfoList,
                                                    boolean bindMountAppsData,
                                                    boolean bindMountAppStorageDirs,
                                                    @Nullable String[] extraArgs)
                                                    throws ZygoteStartFailedEx {
    ArrayList<String> argsForZygote = new ArrayList<>();

    // --runtime-args, --setuid=, --setgid=,
    // and --setgroups= must go first
    argsForZygote.add("--runtime-args");
    argsForZygote.add("--setuid=" + uid);
    argsForZygote.add("--setgid=" + gid);
    argsForZygote.add("--runtime-flags=" + runtimeFlags);
    if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
        argsForZygote.add("--mount-external-default");
    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_INSTALLER) {
        argsForZygote.add("--mount-external-installer");
    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_PASS_THROUGH) {
        argsForZygote.add("--mount-external-pass-through");
    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_ANDROID_WRITABLE) {
        argsForZygote.add("--mount-external-android-writable");
    }

    argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

    // --setgroups is a comma-separated list
    if (gids != null && gids.length > 0) {
        final StringBuilder sb = new StringBuilder();
        sb.append("--setgroups=");

        final int sz = gids.length;
        for (int i = 0; i < sz; i++) {
            if (i != 0) {
                sb.append(',');
            }
            sb.append(gids[i]);
        }

        argsForZygote.add(sb.toString());
    }

    if (niceName != null) {
        argsForZygote.add("--nice-name=" + niceName);
    }

    if (seInfo != null) {
        argsForZygote.add("--seinfo=" + seInfo);
    }

    if (instructionSet != null) {
        argsForZygote.add("--instruction-set=" + instructionSet);
    }

    if (appDataDir != null) {
        argsForZygote.add("--app-data-dir=" + appDataDir);
    }

    if (invokeWith != null) {
        argsForZygote.add("--invoke-with");
        argsForZygote.add(invokeWith);
    }

    if (startChildZygote) {
        argsForZygote.add("--start-child-zygote");
    }

    if (packageName != null) {
        argsForZygote.add("--package-name=" + packageName);
    }

    if (isTopApp) {
        argsForZygote.add(Zygote.START_AS_TOP_APP_ARG);
    }
    if (pkgDataInfoMap != null && pkgDataInfoMap.size() > 0) {
        StringBuilder sb = new StringBuilder();
        sb.append(Zygote.PKG_DATA_INFO_MAP);
        sb.append("=");
        boolean started = false;
        for (Map.Entry<String, Pair<String, Long>> entry : pkgDataInfoMap.entrySet()) {
            if (started) {
                sb.append(',');
            }
            started = true;
            sb.append(entry.getKey());
            sb.append(',');
            sb.append(entry.getValue().first);
            sb.append(',');
            sb.append(entry.getValue().second);
        }
        argsForZygote.add(sb.toString());
    }
    if (allowlistedDataInfoList != null && allowlistedDataInfoList.size() > 0) {
        StringBuilder sb = new StringBuilder();
        sb.append(Zygote.ALLOWLISTED_DATA_INFO_MAP);
        sb.append("=");
        boolean started = false;
        for (Map.Entry<String, Pair<String, Long>> entry : allowlistedDataInfoList.entrySet()) {
            if (started) {
                sb.append(',');
            }
            started = true;
            sb.append(entry.getKey());
            sb.append(',');
            sb.append(entry.getValue().first);
            sb.append(',');
            sb.append(entry.getValue().second);
        }
        argsForZygote.add(sb.toString());
    }

    if (bindMountAppStorageDirs) {
        argsForZygote.add(Zygote.BIND_MOUNT_APP_STORAGE_DIRS);
    }

    if (bindMountAppsData) {
        argsForZygote.add(Zygote.BIND_MOUNT_APP_DATA_DIRS);
    }

    if (disabledCompatChanges != null && disabledCompatChanges.length > 0) {
        StringBuilder sb = new StringBuilder();
        sb.append("--disabled-compat-changes=");

        int sz = disabledCompatChanges.length;
        for (int i = 0; i < sz; i++) {
            if (i != 0) {
                sb.append(',');
            }
            sb.append(disabledCompatChanges[i]);
        }

        argsForZygote.add(sb.toString());
    }

    argsForZygote.add(processClass);

    if (extraArgs != null) {
        Collections.addAll(argsForZygote, extraArgs);
    }

    synchronized(mLock) {
        // The USAP pool can not be used if the application will not use the systems graphics
        // driver.  If that driver is requested use the Zygote application start path.
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                                            zygotePolicyFlags,
                                            argsForZygote);
    }
}
```

这个方法的功能就很简单了，就是将各种参数拼装起来，然后调用`zygoteSendArgsAndGetResult`方法

我们先看`openZygoteSocketIfNeeded`这个方法，它返回了一个`ZygoteState`对象，这个类是对与`ZygoteServerSocket`建立连接后的封装

```java
private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
    try {
        //尝试连接主ZygoteServerSocket
        attemptConnectionToPrimaryZygote();

        //主zygote进程支持此abi
        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        if (mZygoteSecondarySocketAddress != null) {
            // The primary zygote didn't match. Try the secondary.
            //尝试连接辅ZygoteServerSocket
            attemptConnectionToSecondaryZygote();

            //辅zygote进程支持此abi
            if (secondaryZygoteState.matches(abi)) {
                return secondaryZygoteState;
            }
        }
    } catch (IOException ioe) {
        throw new ZygoteStartFailedEx("Error connecting to zygote", ioe);
    }

    throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
}
```

`attemptConnectionToxxxZygote`方法使用`LocalSocket`进行连接，并返回一个`ZygoteState`封装对象

我们之前在 [Android源码分析 - Zygote进程](https://juejin.cn/post/7051507161955827720) 中说过，一般，64位的cpu会启动两个`zygoto`进程，一个64位（主`zygote`），一个32位（辅`zygote`）

![zygote]()

接下来我们看`zygoteSendArgsAndGetResult`方法

```java
private Process.ProcessStartResult zygoteSendArgsAndGetResult(
        ZygoteState zygoteState, int zygotePolicyFlags, @NonNull ArrayList<String> args)
        throws ZygoteStartFailedEx {

    ...

    /*
        * See com.android.internal.os.ZygoteArguments.parseArgs()
        * Presently the wire format to the zygote process is:
        * a) a count of arguments (argc, in essence)
        * b) a number of newline-separated argument strings equal to count
        *
        * After the zygote process reads these it will write the pid of
        * the child or -1 on failure, followed by boolean to
        * indicate whether a wrapper process was used.
        */
    //构建出符合zygote解析规则的参数（argc + argv）
    String msgStr = args.size() + "\n" + String.join("\n", args) + "\n";

    //USAP机制
    if (shouldAttemptUsapLaunch(zygotePolicyFlags, args)) {
        try {
            return attemptUsapSendArgsAndGetResult(zygoteState, msgStr);
        } catch (IOException ex) {
            // If there was an IOException using the USAP pool we will log the error and
            // attempt to start the process through the Zygote.
            Log.e(LOG_TAG, "IO Exception while communicating with USAP pool - "
                    + ex.getMessage());
        }
    }

    return attemptZygoteSendArgsAndGetResult(zygoteState, msgStr);
}
```

`USAP`机制我们先跳过，这个方法