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

## 准备ProcessRecord

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
    //将ProcessRecord添加到待启动列表中
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

## startProcess

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

### 向zygoto发送socket请求

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

`USAP`机制我们先跳过，这个方法就做了一件事：拼装参数，然后调用`attemptZygoteSendArgsAndGetResult`方法

```java
private Process.ProcessStartResult attemptZygoteSendArgsAndGetResult(
        ZygoteState zygoteState, String msgStr) throws ZygoteStartFailedEx {
    try {
        final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
        final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;

        zygoteWriter.write(msgStr);
        zygoteWriter.flush();

        // Always read the entire result from the input stream to avoid leaving
        // bytes in the stream for future process starts to accidentally stumble
        // upon.
        Process.ProcessStartResult result = new Process.ProcessStartResult();
        result.pid = zygoteInputStream.readInt();
        result.usingWrapper = zygoteInputStream.readBoolean();

        if (result.pid < 0) {
            throw new ZygoteStartFailedEx("fork() failed");
        }

        return result;
    } catch (IOException ex) {
        zygoteState.close();
        Log.e(LOG_TAG, "IO Exception while communicating with Zygote - "
                + ex.toString());
        throw new ZygoteStartFailedEx(ex);
    }
}
```

这个方法很明显就能看出来，这是一次`socket`通信发送 -> 接收

具体`zygote`进程接收到`socket`后做了什么可以回顾我之前写的文章 [Android源码分析 - Zygote进程](https://juejin.cn/post/7051507161955827720#heading-29)

## handleProcessStartedLocked

向`zygote`发送完`socket`请求后，`zygote`开始`fork`App进程，`fork`完后会将App进程的`pid`和`usingWrapper`信息再通过`socket`传回`system_server`，此时程序会继续执行`handleProcessStartedLocked`方法

```java
boolean handleProcessStartedLocked(ProcessRecord app, int pid, boolean usingWrapper,
        long expectedStartSeq, boolean procAttached) {
    //从待启动列表中移除此ProcessRecord
    mPendingStarts.remove(expectedStartSeq);
    final String reason = isProcStartValidLocked(app, expectedStartSeq);
    //未通过进程启动验证，杀死进程
    if (reason != null) {
        app.pendingStart = false;
        killProcessQuiet(pid);
        Process.killProcessGroup(app.uid, app.pid);
        noteAppKill(app, ApplicationExitInfo.REASON_OTHER,
                ApplicationExitInfo.SUBREASON_INVALID_START, reason);
        return false;
    }
    
    ... //记录进程启动

    //通知看门狗有进程启动
    Watchdog.getInstance().processStarted(app.processName, pid);

    ... //记录进程启动

    //设置ProcessRecord
    app.setPid(pid);
    app.setUsingWrapper(usingWrapper);
    app.pendingStart = false;
    //从PidMap中获取未清理的ProcessRecord
    ProcessRecord oldApp;
    synchronized (mService.mPidsSelfLocked) {
        oldApp = mService.mPidsSelfLocked.get(pid);
    }
    // If there is already an app occupying that pid that hasn't been cleaned up
    //清理ProcessRecord
    if (oldApp != null && !app.isolated) {
        mService.cleanUpApplicationRecordLocked(oldApp, false, false, -1,
                true /*replacingPid*/);
    }
    //将ProcessRecord添加到PidMap中
    mService.addPidLocked(app);
    synchronized (mService.mPidsSelfLocked) {
        //attach超时检测
        if (!procAttached) {
            Message msg = mService.mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
            msg.obj = app;
            mService.mHandler.sendMessageDelayed(msg, usingWrapper
                    ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
        }
    }
    return true;
}
```

这个方法将从`zygote` `fork`后得到的信息设置到`ProcessRecord`中，然后将此`ProcessRecord`添加到`PidMap`中（`AMS.mPidsSelfLocked`），后续当`attachApplication`时会用到它

# ActivityThread

`zygote`进程将App进程`fork`出来后，便通过反射调用我们之前设置的`entryPoint`类的`main`方法，即`android.app.ActivityThread.main(String[] args)`方法

```java
public static void main(String[] args) {
    // Install selective syscall interception
    //设置拦截器，拦截部分系统调用自行处理
    AndroidOs.install();

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    //资源关闭检测器
    CloseGuard.setEnabled(false);

    //初始化用户环境
    Environment.initForCurrentUser();

    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    //设置CA证书搜索位置
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    // Call per-process mainline module initialization.
    //初始化主模块各个注册服务
    initializeMainlineModules();

    //预设进程名
    Process.setArgV0("<pre-initialized>");

    //准备Looper
    Looper.prepareMainLooper();

    // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
    // It will be in the format "seq=114"
    //查找startSeq参数
    long startSeq = 0;
    if (args != null) {
        for (int i = args.length - 1; i >= 0; --i) {
            if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                startSeq = Long.parseLong(
                        args[i].substring(PROC_START_SEQ_IDENT.length()));
            }
        }
    }
    //创建App进程ActivityThread实例
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    //设置全局Handler
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    //Looper循环处理消息
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

在`main`方法中主要做了两件事，一是启动`Looper`，循环处理消息，保证进程不会退出，二是实例化`ActivityThread`并执行`attach`方法

```java
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mConfigurationController = new ConfigurationController(this);
    mSystemThread = system;
    if (!system) {    //非系统ActivityThread
        //预设进程名
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                UserHandle.myUserId());
        //处理一些错误异常需要使用ActivityThread，将其传入
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        //AMS代理binder对象
        final IActivityManager mgr = ActivityManager.getService();
        try {
            //执行AMS.attachApplication方法
            mgr.attachApplication(mAppThread, startSeq);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        // Watch for getting close to heap limit.
        //每次GC时检测内存，如果内存不足则会尝试释放部分不可见的Activity
        BinderInternal.addGcWatcher(new Runnable() {
            @Override public void run() {
                if (!mSomeActivitiesChanged) {
                    return;
                }
                Runtime runtime = Runtime.getRuntime();
                long dalvikMax = runtime.maxMemory();
                long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                if (dalvikUsed > ((3*dalvikMax)/4)) {
                    if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                            + " total=" + (runtime.totalMemory()/1024)
                            + " used=" + (dalvikUsed/1024));
                    mSomeActivitiesChanged = false;
                    try {
                        ActivityTaskManager.getService().releaseSomeActivities(mAppThread);
                    } catch (RemoteException e) {
                        throw e.rethrowFromSystemServer();
                    }
                }
            }
        });
    } else {    //系统ActivityThread
        ...
    }

    //处理ConfigChanged相关逻辑（屏幕旋转之类）
    ViewRootImpl.ConfigChangedCallback configChangedCallback = (Configuration globalConfig) -> {
        synchronized (mResourcesManager) {
            // We need to apply this change to the resources immediately, because upon returning
            // the view hierarchy will be informed about it.
            if (mResourcesManager.applyConfigurationToResources(globalConfig,
                    null /* compat */)) {
                mConfigurationController.updateLocaleListFromAppContext(
                        mInitialApplication.getApplicationContext());

                // This actually changed the resources! Tell everyone about it.
                final Configuration updatedConfig =
                        mConfigurationController.updatePendingConfiguration(globalConfig);
                if (updatedConfig != null) {
                    sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
                    mPendingConfiguration = updatedConfig;
                }
            }
        }
    };
    ViewRootImpl.addConfigCallback(configChangedCallback);
}
```

而`attach`方法最重要的一步又是调用了`AMS`的`attachApplication`方法

# AMS.attachApplication

```java
public final void attachApplication(IApplicationThread thread, long startSeq) {
    if (thread == null) {
        throw new SecurityException("Invalid application interface");
    }
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid, callingUid, startSeq);
        Binder.restoreCallingIdentity(origId);
    }
}

private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {

    // Find the application record that is being attached...  either via
    // the pid if we are running in multiple processes, or just pull the
    // next app record if we are emulating process with anonymous threads.
    ProcessRecord app;
    long startTime = SystemClock.uptimeMillis();
    long bindApplicationTimeMillis;
    if (pid != MY_PID && pid >= 0) {
        //通过pid查找PidMap中存在的ProcessRecord
        //对应着handleProcessStartedLocked方法中执行的中的mService.addPidLocked方法
        //在进程同步启动模式下，这里应该是必能取到的
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
        //如果此ProcessRecord对不上App的ProcessRecord，则将其清理掉
        if (app != null && (app.startUid != callingUid || app.startSeq != startSeq)) {
            ...
            // If there is already an app occupying that pid that hasn't been cleaned up
            cleanUpApplicationRecordLocked(app, false, false, -1,
                        true /*replacingPid*/);
            removePidLocked(app);
            app = null;
        }
    } else {
        app = null;
    }

    // It's possible that process called attachApplication before we got a chance to
    // update the internal state.
    //在进程异步启动模式下，有可能尚未执行到handleProcessStartedLocked方法
    //所以从PidMap中无法取到相应的ProcessRecord
    //这时候从ProcessList.mPendingStarts这个待启动列表中获取ProcessRecord
    if (app == null && startSeq > 0) {
        final ProcessRecord pending = mProcessList.mPendingStarts.get(startSeq);
        if (pending != null && pending.startUid == callingUid && pending.startSeq == startSeq
                && mProcessList.handleProcessStartedLocked(pending, pid, pending
                        .isUsingWrapper(),
                        startSeq, true)) {
            app = pending;
        }
    }

    //没有找到相应的ProcessRecord，杀死进程
    if (app == null) {
        if (pid > 0 && pid != MY_PID) {
            killProcessQuiet(pid);
        } else {
            try {
                thread.scheduleExit();
            } catch (Exception e) {
                // Ignore exceptions.
            }
        }
        return false;
    }

    // If this application record is still attached to a previous
    // process, clean it up now.
    //如果ProcessRecord绑定了其他的ApplicationThread，则需要清理这个进程
    if (app.thread != null) {
        handleAppDiedLocked(app, true, true);
    }

    final String processName = app.processName;
    try {
        //注册App进程死亡回调
        AppDeathRecipient adr = new AppDeathRecipient(
                app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        //如果出现异常则重启进程
        app.resetPackageList(mProcessStats);
        mProcessList.startProcessLocked(app,
                new HostingRecord("link fail", processName),
                ZYGOTE_POLICY_FLAG_EMPTY);
        return false;
    }

    //初始化ProcessRecord各参数
    app.curAdj = app.setAdj = app.verifiedAdj = ProcessList.INVALID_ADJ;
    mOomAdjuster.setAttachingSchedGroupLocked(app);
    app.forcingToImportant = null;
    updateProcessForegroundLocked(app, false, 0, false);
    app.hasShownUi = false;
    app.setDebugging(false);
    app.setCached(false);
    app.killedByAm = false;
    app.killed = false;


    // We carefully use the same state that PackageManager uses for
    // filtering, since we use this flag to decide if we need to install
    // providers when user is unlocked later
    app.unlocked = StorageManager.isUserKeyUnlocked(app.userId);

    //移除之前在handleProcessStartedLocked中设置的attach超时检测
    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

    //普通App启动肯定在system_server准备完成后，所以此处为true
    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;
    //设置ContentProvider启动超时检测
    if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
        Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
        msg.obj = app;
        mHandler.sendMessageDelayed(msg,
                ContentResolver.CONTENT_PROVIDER_PUBLISH_TIMEOUT_MILLIS);
    }

    final BackupRecord backupTarget = mBackupTargets.get(app.userId);
    try {
        //对应着开发者模式里的 Select debug app 和 Wait for debugger
        int testMode = ApplicationThreadConstants.DEBUG_OFF;
        if (mDebugApp != null && mDebugApp.equals(processName)) {
            testMode = mWaitForDebugger
                ? ApplicationThreadConstants.DEBUG_WAIT
                : ApplicationThreadConstants.DEBUG_ON;
            app.setDebugging(true);
            if (mDebugTransient) {
                mDebugApp = mOrigDebugApp;
                mWaitForDebugger = mOrigWaitForDebugger;
            }
        }

        boolean enableTrackAllocation = false;
        if (mTrackAllocationApp != null && mTrackAllocationApp.equals(processName)) {
            enableTrackAllocation = true;
            mTrackAllocationApp = null;
        }

        // If the app is being launched for restore or full backup, set it up specially
        boolean isRestrictedBackupMode = false;
        ... //备份相关

        final ActiveInstrumentation instr;
        ... //自动化测试相关

        ApplicationInfo appInfo = instr != null ? instr.mTargetInfo : app.info;
        app.compat = compatibilityInfoForPackage(appInfo);

        ProfilerInfo profilerInfo = null;
        String preBindAgent = null;
        ... //性能分析相关

        // We deprecated Build.SERIAL and it is not accessible to
        // Instant Apps and target APIs higher than O MR1. Since access to the serial
        // is now behind a permission we push down the value.
        //序列号（Android 8.0后不可再通过Build.SERIAL获取序列号）
        final String buildSerial = (!appInfo.isInstantApp()
                && appInfo.targetSdkVersion < Build.VERSION_CODES.P)
                        ? sTheRealBuildSerial : Build.UNKNOWN;

        
        ... //自动化测试相关
        ... //性能分析相关
        
        //debug模式
        if ((app.info.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0) {
            thread.attachStartupAgents(app.info.dataDir);
        }

        ... //自动填充功能（账号密码等）
        ... //内容捕获相关（ContentCaptureManager）

        //自动化测试
        final ActiveInstrumentation instr2 = app.getActiveInstrumentation();
        if (mPlatformCompat != null) {
            mPlatformCompat.resetReporting(app.info);
        }
        final ProviderInfoList providerList = ProviderInfoList.fromList(providers);
        //调用ApplicationThread.bindApplication方法
        if (app.isolatedEntryPoint != null) {
            // This is an isolated process which should just call an entry point instead of
            // being bound to an application.
            thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
        } else if (instr2 != null) {
            thread.bindApplication(processName, appInfo, providerList,
                    instr2.mClass,
                    profilerInfo, instr2.mArguments,
                    instr2.mWatcher,
                    instr2.mUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.isPersistent(),
                    new Configuration(app.getWindowProcessController().getConfiguration()),
                    app.compat, getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked(),
                    buildSerial, autofillOptions, contentCaptureOptions,
                    app.mDisabledCompatChanges);
        } else {
            thread.bindApplication(processName, appInfo, providerList, null, profilerInfo,
                    null, null, null, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.isPersistent(),
                    new Configuration(app.getWindowProcessController().getConfiguration()),
                    app.compat, getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked(),
                    buildSerial, autofillOptions, contentCaptureOptions,
                    app.mDisabledCompatChanges);
        }
        ...

        // Make app active after binding application or client may be running requests (e.g
        // starting activities) before it is ready.
        //ProcessRecord保存ApplicationThread代理对象
        app.makeActive(thread, mProcessStats);
        //更新进程使用情况
        mProcessList.updateLruProcessLocked(app, false, null);
        app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
    } catch (Exception e) {
        //出现错误，杀死进程
        app.resetPackageList(mProcessStats);
        app.unlinkDeathRecipient();
        app.kill("error during bind", ApplicationExitInfo.REASON_INITIALIZATION_FAILURE, true);
        handleAppDiedLocked(app, false, true);
        return false;
    }

    // Remove this record from the list of starting applications.
    //从persistent启动列表中移除此ProcessRecord
    //persistent是manifest中application标签下的一个属性
    //设置了此属性代表此App会跟随系统启动而启动
    mPersistentStartingProcesses.remove(app);

    boolean badApp = false;
    boolean didSomething = false;

    // See if the top visible activity is waiting to run in this process...
    //检查是否有Activity等待启动
    if (normalMode) {
        try {
            didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
        } catch (Exception e) {
            badApp = true;
        }
    }

    // Find any services that should be running in this process...
    //检查是否有Services等待启动
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
        } catch (Exception e) {
            badApp = true;
        }
    }

    // Check if a next-broadcast receiver is in this process...
    //检查是否有广播接收器需要启动
    if (!badApp && isPendingBroadcastProcessLocked(pid)) {
        try {
            didSomething |= sendPendingBroadcastsLocked(app);
        } catch (Exception e) {
            // If the app died trying to launch the receiver we declare it 'bad'
            badApp = true;
        }
    }

    ... //备份相关

    //以上几步发生异常，杀死App进程
    if (badApp) {
        app.kill("error during init", ApplicationExitInfo.REASON_INITIALIZATION_FAILURE, true);
        handleAppDiedLocked(app, false, true);
        return false;
    }

    if (!didSomething) {
        //更新进程OOM等级
        updateOomAdjLocked(app, OomAdjuster.OOM_ADJ_REASON_PROCESS_BEGIN);
    }

    return true;
}
```

总结一下这个方法主要做了哪些事，首先获取`ProcessRecord`，然后对其做一些初始化设置，然后调用`ApplicaionThread.bindApplication`方法，最后分别检查处理`Activity`、`Service`和`BroadcastReceiver`的启动

## 获取ProcessRecord

我们看一下这个方法是怎么获取`ProcessRecord`的，我们先回顾一下之前在`startProcessLocked`方法的最后，会使用同步或异步的方式启动进程，最终两者都会调用`startProcess`和`handleProcessStartedLocked`方法

### 同步启动进程

我们回顾一下之前讲到的`ActivityManagerInternal.startProcess`方法，可以发现它内部使用了`synchronized (ActivityManagerService.this)`加锁，而`AMS.attachApplication`方法同样也使用了`AMS`实例对象加了锁，所以在同步启动进程的情况下，必然会先执行`handleProcessStartedLocked`方法，再执行`attachApplication`方法，根据之前所分析的，`handleProcessStartedLocked`方法会将`ProcessRecord`存到PidMap中，然后`attachApplication`方法又会从PidMap中去取，此时取出的`ProcessRecord`必然不为`null`

### 异步启动进程

在异步启动进程的情况下，是通过`Handler`将启动进程的工作插入到任务队列中，这个任务的执行是不在锁的作用域范围内的，在这个任务内没有对`startProcess`方法加锁，只对`handleProcessStartedLocked`方法加了锁，所以这里会有两种情况：

- 先执行`handleProcessStartedLocked`方法，再执行`attachApplication`方法

    这种情况和同步启动进程的执行顺序是一样的，`ProcessRecord`获取方式也相同

- 先执行`attachApplication`方法，再执行`handleProcessStartedLocked`方法

    这种情况下，PidMap中取不到相应的`ProcessRecord`，此时`ProcessList.mPendingStarts`中还没有将`ProcessRecord`移除，所以会从`mPendingStarts`这个启动列表中取出`ProcessRecord`，然后再调用`handleProcessStartedLocked`方法，等到`attachApplication`方法走完，锁释放后，在进入到外部的`handleProcessStartedLocked`重载方法，这个方法会先判断`mPendingStarts`中是否还存在对应的`ProcessRecord`，如果不存在，便会直接返回，保证`handleProcessStartedLocked`方法只执行一次

# ApplicationThread.bindApplication

接着，我们继续看重点方法`ApplicationThread.bindApplication`

`ApplicationThread`是`ActivityThread`的一个内部类

```java
@Override
public final void bindApplication(String processName, ApplicationInfo appInfo,
        ProviderInfoList providerList, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableBinderTracking, boolean trackAllocation,
        boolean isRestrictedBackupMode, boolean persistent, Configuration config,
        CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
        String buildSerial, AutofillOptions autofillOptions,
        ContentCaptureOptions contentCaptureOptions, long[] disabledCompatChanges) {
    if (services != null) {
        // Setup the service cache in the ServiceManager
        //初始化通用系统服务缓存
        ServiceManager.initServiceCache(services);
    }

    setCoreSettings(coreSettings);

    AppBindData data = new AppBindData();
    data.processName = processName;
    data.appInfo = appInfo;
    data.providers = providerList.getList();
    data.instrumentationName = instrumentationName;
    data.instrumentationArgs = instrumentationArgs;
    data.instrumentationWatcher = instrumentationWatcher;
    data.instrumentationUiAutomationConnection = instrumentationUiConnection;
    data.debugMode = debugMode;
    data.enableBinderTracking = enableBinderTracking;
    data.trackAllocation = trackAllocation;
    data.restrictedBackupMode = isRestrictedBackupMode;
    data.persistent = persistent;
    data.config = config;
    data.compatInfo = compatInfo;
    data.initProfilerInfo = profilerInfo;
    data.buildSerial = buildSerial;
    data.autofillOptions = autofillOptions;
    data.contentCaptureOptions = contentCaptureOptions;
    data.disabledCompatChanges = disabledCompatChanges;
    sendMessage(H.BIND_APPLICATION, data);
}
```

这个方法很简单，只是将参数包装成一个`AppBindData`，然后通过`Handler`发送消息处理，根据消息的类型，最终会调用`ActivityThread.handleBindApplication`方法

```java
private void handleBindApplication(AppBindData data) {
    // Register the UI Thread as a sensitive thread to the runtime.
    //将UI线程注册成JIT敏感线程
    VMRuntime.registerSensitiveThread();

    ...

    ... //AppCompat相关

    mBoundApplication = data;
    
    ... //Configuration相关

    mProfiler = new Profiler();
    ... //性能分析相关

    // send up app name; do this *before* waiting for debugger
    //设置进程名
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName,
                                            data.appInfo.packageName,
                                            UserHandle.myUserId());
    VMRuntime.setProcessPackageName(data.appInfo.packageName);

    // Pass data directory path to ART. This is used for caching information and
    // should be set before any application code is loaded.
    //设置进程数据目录
    VMRuntime.setProcessDataDirectory(data.appInfo.dataDir);

    //性能分析相关
    if (mProfiler.profileFd != null) {
        mProfiler.startProfiling();
    }

    // If the app is Honeycomb MR1 or earlier, switch its AsyncTask
    // implementation to use the pool executor.  Normally, we use the
    // serialized executor as the default. This has to happen in the
    // main thread so the main looper is set right.
    //当App的targetSdkVersion小于等于 3.1 (12) 时，AsyncTask使用线程池实现
    if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
        AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
    }

    // Let the util.*Array classes maintain "undefined" for apps targeting Pie or earlier.
    //当App的targetSdkVersion大于等于 10 (29) 时，针对Android SDK提供的容器（SparseArray等）
    //如果index越界，会主动抛ArrayIndexOutOfBoundsException异常
    //（之前数组越界的行为未被定义）
    UtilConfig.setThrowExceptionForUpperArrayOutOfBounds(
            data.appInfo.targetSdkVersion >= Build.VERSION_CODES.Q);

    //当App的targetSdkVersion大于等于 5.0 (21) 时，回收正在使用的Message会抛出异常
    Message.updateCheckRecycle(data.appInfo.targetSdkVersion);

    // Supply the targetSdkVersion to the UI rendering module, which may
    // need it in cases where it does not have access to the appInfo.
    //设置App目标版本
    android.graphics.Compatibility.setTargetSdkVersion(data.appInfo.targetSdkVersion);

    /*
        * Before spawning a new process, reset the time zone to be the system time zone.
        * This needs to be done because the system time zone could have changed after the
        * the spawning of this process. Without doing this this process would have the incorrect
        * system time zone.
        */
    //设置时区
    TimeZone.setDefault(null);

    /*
        * Set the LocaleList. This may change once we create the App Context.
        */
    LocaleList.setDefault(data.config.getLocales());

    //加载系统字体
    if (Typeface.ENABLE_LAZY_TYPEFACE_INITIALIZATION) {
        try {
            Typeface.setSystemFontMap(data.mSerializedSystemFontMap);
        } catch (IOException | ErrnoException e) {
            Typeface.loadPreinstalledSystemFontMap();
        }
    }

    //更新Configuration
    synchronized (mResourcesManager) {
        /*
            * Update the system configuration since its preloaded and might not
            * reflect configuration changes. The configuration object passed
            * in AppBindData can be safely assumed to be up to date
            */
        mResourcesManager.applyConfigurationToResources(data.config, data.compatInfo);
        mCurDefaultDisplayDpi = data.config.densityDpi;

        // This calls mResourcesManager so keep it within the synchronized block.
        mConfigurationController.applyCompatConfiguration();
    }

    final boolean isSdkSandbox = data.sdkSandboxClientAppPackage != null;
    //获取LoadedApk
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo, isSdkSandbox);
    //沙盒模式
    if (isSdkSandbox) {
        data.info.setSdkSandboxStorage(data.sdkSandboxClientAppVolumeUuid,
                data.sdkSandboxClientAppPackage);
    }

    //性能分析器代理JVM（JVMTI）
    if (agent != null) {
        handleAttachAgent(agent, data.info);
    }

    /**
        * Switch this process to density compatibility mode if needed.
        */
    //在manifest，supports-screens标签中设置了android:anyDensity
    //详见：https://developer.android.com/guide/topics/manifest/supports-screens-element#any
    if ((data.appInfo.flags&ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
            == 0) {
        //指示App包含用于适应任何屏幕密度的资源
        mDensityCompatMode = true;
        Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
    }
    //设置默认密度
    mConfigurationController.updateDefaultDensity(data.config.densityDpi);

    /* 设置 12/24 小时时间制 */
    // mCoreSettings is only updated from the main thread, while this function is only called
    // from main thread as well, so no need to lock here.
    final String use24HourSetting = mCoreSettings.getString(Settings.System.TIME_12_24);
    Boolean is24Hr = null;
    if (use24HourSetting != null) {
        is24Hr = "24".equals(use24HourSetting) ? Boolean.TRUE : Boolean.FALSE;
    }
    // null : use locale default for 12/24 hour formatting,
    // false : use 12 hour format,
    // true : use 24 hour format.
    DateFormat.set24HourTimePref(is24Hr);

    //更新view debug属性sDebugViewAttributes
    //设置了这个属性，View将会保存它本身的属性
    //和Layout Inspector相关
    updateDebugViewAttributeState();

    //初始化默认线程策略
    StrictMode.initThreadDefaults(data.appInfo);
    //初始化默认VM策略
    StrictMode.initVmDefaults(data.appInfo);

    //debug模式
    if (data.debugMode != ApplicationThreadConstants.DEBUG_OFF) {
        // XXX should have option to change the port.
        Debug.changeDebugPort(8100);
        if (data.debugMode == ApplicationThreadConstants.DEBUG_WAIT) {
            Slog.w(TAG, "Application " + data.info.getPackageName()
                    + " is waiting for the debugger on port 8100...");

            IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.showWaitingForDebugger(mAppThread, true);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }

            Debug.waitForDebugger();

            try {
                mgr.showWaitingForDebugger(mAppThread, false);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }

        } else {
            Slog.w(TAG, "Application " + data.info.getPackageName()
                    + " can be debugged on port 8100...");
        }
    }

    // Allow binder tracing, and application-generated systrace messages if we're profileable.
    //debug模式
    boolean isAppDebuggable = (data.appInfo.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
    //性能分析模式
    boolean isAppProfileable = isAppDebuggable || data.appInfo.isProfileable();
    //允许应用程序跟踪
    Trace.setAppTracingAllowed(isAppProfileable);
    //开启Binder栈追踪
    if ((isAppProfileable || Build.IS_DEBUGGABLE) && data.enableBinderTracking) {
        Binder.enableStackTracking();
    }

    // Initialize heap profiling.
    //初始化堆分析
    if (isAppProfileable || Build.IS_DEBUGGABLE) {
        nInitZygoteChildHeapProfiling();
    }

    // Allow renderer debugging features if we're debuggable.
    //开启硬件加速调试功能
    HardwareRenderer.setDebuggingEnabled(isAppDebuggable || Build.IS_DEBUGGABLE);
    HardwareRenderer.setPackageName(data.appInfo.packageName);

    // Pass the current context to HardwareRenderer
    //为硬件加速设置Context
    HardwareRenderer.setContextForInit(getSystemContext());

    // Instrumentation info affects the class loader, so load it before
    // setting up the app context.
    //准备自动化测试信息
    final InstrumentationInfo ii;
    if (data.instrumentationName != null) {
        ii = prepareInstrumentation(data);
    } else {
        ii = null;
    }

    //创建Context
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    //更新区域列表
    mConfigurationController.updateLocaleListFromAppContext(appContext);

    // Initialize the default http proxy in this process.
    // In pre-boot mode (doing initial launch to collect password), not all system is up.
    // This includes the connectivity service, so trying to obtain ConnectivityManager at
    // that point would return null. Check whether the ConnectivityService is available, and
    // avoid crashing with a NullPointerException if it is not.
    //设置默认HTTP代理
    final IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
    if (b != null) {
        final ConnectivityManager cm =
                appContext.getSystemService(ConnectivityManager.class);
        Proxy.setHttpProxyConfiguration(cm.getDefaultProxy());
    }

    if (!Process.isIsolated()) {
        final int oldMask = StrictMode.allowThreadDiskWritesMask();
        try {
            setupGraphicsSupport(appContext);
        } finally {
            StrictMode.setThreadPolicyMask(oldMask);
        }
    } else {
        HardwareRenderer.setIsolatedProcess(true);
    }

    // Install the Network Security Config Provider. This must happen before the application
    // code is loaded to prevent issues with instances of TLS objects being created before
    // the provider is installed.
    //网络安全设置
    NetworkSecurityConfigProvider.install(appContext);

    // For backward compatibility, TrafficStats needs static access to the application context.
    // But for isolated apps which cannot access network related services, service discovery
    // is restricted. Hence, calling this would result in NPE.
    //初始化流量统计工具
    if (!Process.isIsolated()) {
        TrafficStats.init(appContext);
    }

    // Continue loading instrumentation.
    if (ii != null) {
        //如果设置了自动化测试，实例化指定的自动化测试类
        initInstrumentation(ii, data, appContext);
    } else {
        //直接实例化Instrumentation
        mInstrumentation = new Instrumentation();
        mInstrumentation.basicInit(this);
    }

    //调整应用可用内存上限
    if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
    } else {
        // Small heap, clamp to the current growth limit and let the heap release
        // pages after the growth limit to the non growth limit capacity. b/18387825
        dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
    }

    // Allow disk access during application and provider setup. This could
    // block processing ordered broadcasts, but later processing would
    // probably end up doing the same disk access.
    Application app;
    final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
    final StrictMode.ThreadPolicy writesAllowedPolicy = StrictMode.getThreadPolicy();
    try {
        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        //创建Application
        app = data.info.makeApplicationInner(data.restrictedBackupMode, null);

        // Propagate autofill compat state
        //设置自动填充功能
        app.setAutofillOptions(data.autofillOptions);

        // Propagate Content Capture options
        //设置内容捕获功能
        app.setContentCaptureOptions(data.contentCaptureOptions);
        sendMessage(H.SET_CONTENT_CAPTURE_OPTIONS_CALLBACK, data.appInfo.packageName);

        mInitialApplication = app;
        //更新HTTP代理
        final boolean updateHttpProxy;
        synchronized (this) {
            updateHttpProxy = mUpdateHttpProxyOnBind;
            // This synchronized block ensures that any subsequent call to updateHttpProxy()
            // will see a non-null mInitialApplication.
        }
        if (updateHttpProxy) {
            ActivityThread.updateHttpProxy(app);
        }

        // don't bring up providers in restricted mode; they may depend on the
        // app's custom Application class
        //在非受限模式下启动ContentProvider
        if (!data.restrictedBackupMode) {
            if (!ArrayUtils.isEmpty(data.providers)) {
                installContentProviders(app, data.providers);
            }
        }

        // Do this after providers, since instrumentation tests generally start their
        // test thread at this point, and we don't want that racing.
        //执行onCreate方法（默认Instrumentation实现为空方法）
        mInstrumentation.onCreate(data.instrumentationArgs);
        //执行Application的onCreate方法
        mInstrumentation.callApplicationOnCreate(app);
    } finally {
        // If the app targets < O-MR1, or doesn't change the thread policy
        // during startup, clobber the policy to maintain behavior of b/36951662
        if (data.appInfo.targetSdkVersion < Build.VERSION_CODES.O_MR1
                || StrictMode.getThreadPolicy().equals(writesAllowedPolicy)) {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }

    // Preload fonts resources
    //预加载字体资源
    FontsContract.setApplicationContextForResources(appContext);
    if (!Process.isIsolated()) {
        try {
            final ApplicationInfo info =
                    getPackageManager().getApplicationInfo(
                            data.appInfo.packageName,
                            PackageManager.GET_META_DATA /*flags*/,
                            UserHandle.myUserId());
            if (info.metaData != null) {
                final int preloadedFontsResource = info.metaData.getInt(
                        ApplicationInfo.METADATA_PRELOADED_FONTS, 0);
                if (preloadedFontsResource != 0) {
                    data.info.getResources().preloadFonts(preloadedFontsResource);
                }
            }
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

这个方法很重要，我们先过一下几个重点部分，然后再按着主线继续往下研究：

- `Debug`、`Profiler`、`Layout Inspector`

  `Android`应用开发的同学对这三样肯定不陌生，在`Android Studio`中我们可以对App进行调试，性能分析和布局检查，在这个方法中，我们可以找到与这三样相关的一些代码

- 获取`LoadedApk`

  `LoadedApk`是`Apk`文件在内存中的表示，包含了`Apk`文件中的代码、资源、组件、`manifest`等信息

- 创建`Context`

  这里通过`ActivityThread`和`LoadedApk`创建出了一个`ContextImpl`

- 实例化`Instrumentation`

  这里和自动化测试相关，如果设置了自动化测试，实例化指定的自动化测试类，否则实例化默认的`Instrumentation`

- 创建`Application`

  这里根据`LoadedApk`创建出相应的`Application`，需要注意，这里创建的`Application`并不与上面创建出的`ContextImpl`绑定，而是在创建`Application`的过程中，以同样的参数重新创建了一个`ContextImpl`，然后调用`attachBaseContext`方法绑定它

- 设置`HTTP`代理

  App在启动过程中设置`HTTP`代理，所以我们在开发过程中使用代理抓包等时候需要注意，设置了代理后需要重启App才会生效

- 启动`ContentProvider`

  `ContentProvider`的启动过程以后会新开文章进行分析，这里只需要知道`ContentProvider`启动的入口在这就行了

- 执行`Application`的`onCreate`方法

  当创建完`Application`，执行`attachBaseContext`方法后，便会调用`onCreate`方法

我们拣重点来看，首先是`Application`的创建过程

在上文的方法中，调用了`data.info.makeApplicationInner`方法创建`Application`，其中`data.info`为`LoadedApk`类型

```java
public Application makeApplicationInner(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    return makeApplicationInner(forceDefaultAppClass, instrumentation,
            /* allowDuplicateInstances= */ false);
}

private Application makeApplicationInner(boolean forceDefaultAppClass,
        Instrumentation instrumentation, boolean allowDuplicateInstances) {
    //如果之前创建过了就可以直接返回
    if (mApplication != null) {
        return mApplication;
    };

    //从缓存中获取Application（多个App可以运行在同一个进程中）
    synchronized (sApplications) {
        final Application cached = sApplications.get(mPackageName);
        if (cached != null) {
            // Looks like this is always happening for the system server, because
            // the LoadedApk created in systemMain() -> attach() isn't cached properly?
            if (!"android".equals(mPackageName)) {
                Slog.wtfStack(TAG, "App instance already created for package=" + mPackageName
                        + " instance=" + cached);
            }
            if (!allowDuplicateInstances) {
                mApplication = cached;
                return cached;
            }
            // Some apps intentionally call makeApplication() to create a new Application
            // instance... Sigh...
        }
    }

    Application app = null;

    final String myProcessName = Process.myProcessName();
    //获取Application类名（App可以自定义Application这个应该所有开发都知道吧）
    //对应AndroidManifest中application标签下的android:name属性
    String appClass = mApplicationInfo.getCustomApplicationClassNameForProcess(
            myProcessName);
    //没有设置自定义Application或强制使用默认Application的情况下，使用默认Application
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        //初始化ContextClassLoader
        final java.lang.ClassLoader cl = getClassLoader();
        if (!mPackageName.equals("android")) {
            initializeJavaContextClassLoader();
        }

        //Android共享库资源ID动态映射
        // Rewrite the R 'constants' for all library apks.
        SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers(
                false, false);
        for (int i = 0, n = packageIdentifiers.size(); i < n; i++) {
            final int id = packageIdentifiers.keyAt(i);
            if (id == 0x01 || id == 0x7f) {
                continue;
            }

            rewriteRValues(cl, packageIdentifiers.valueAt(i), id);
        }

        //创建BaseContext
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        // The network security config needs to be aware of multiple
        // applications in the same process to handle discrepancies
        //网络安全设置
        NetworkSecurityConfigProvider.handleNewApplication(appContext);
        //创建Application
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        if (!mActivityThread.mInstrumentation.onException(app, e)) {
            throw new RuntimeException(
                "Unable to instantiate application " + appClass
                + " package " + mPackageName + ": " + e.toString(), e);
        }
    }
    //将App的Application添加到缓存中（多个App可以运行在同一个进程中）
    mActivityThread.mAllApplications.add(app);
    mApplication = app;
    if (!allowDuplicateInstances) {
        synchronized (sApplications) {
            sApplications.put(mPackageName, app);
        }
    }

    if (instrumentation != null) {
        try {
            //调用Application的OnCreate方法
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!instrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
            }
        }
    }

    return app;
}
```