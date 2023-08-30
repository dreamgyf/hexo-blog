---
title: Android源码分析 - Service启动流程
date: 2023-08-24 16:10:53
tags: 
- Android源码
- Service
- ActivityManagerService
categories: 
- [Android, 源码分析]
- [Android, Service]
- [Android, ActivityManagerService]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

在之前的文章中，我们已经分析过了四大组件中`Activity`和`ContentProvider`的启动流程，这次我们就来讲讲四大组件之一的`Service`是如何启动和绑定的

# 入口

启动`Service`有两种方式，一是`startService`，一是`bindService`，它们的实现都在`ContextImpl`中

## startService

当`Service`通过这种方式启动后，会一直运行下去，直到外部调用了`stopService`或内部调用`stopSelf`

```java
//frameworks/base/core/java/android/app/ContextImpl.java
public ComponentName startService(Intent service) {
    warnIfCallingFromSystemProcess();
    return startServiceCommon(service, false, mUser);
}

private ComponentName startServiceCommon(Intent service, boolean requireForeground,
        UserHandle user) {
    try {
        validateServiceIntent(service);
        service.prepareToLeaveProcess(this);
        //调用AMS.startService
        ComponentName cn = ActivityManager.getService().startService(
                mMainThread.getApplicationThread(), service,
                service.resolveTypeIfNeeded(getContentResolver()), requireForeground,
                getOpPackageName(), getAttributionTag(), user.getIdentifier());
        //通过AMS层返回的ComponentName.packageName来判断是否出错以及错误类型
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException(
                        "Not allowed to start service " + service
                        + " without permission " + cn.getClassName());
            } else if (cn.getPackageName().equals("!!")) {
                throw new SecurityException(
                        "Unable to start service " + service
                        + ": " + cn.getClassName());
            } else if (cn.getPackageName().equals("?")) {
                throw new IllegalStateException(
                        "Not allowed to start service " + service + ": " + cn.getClassName());
            }
        }
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

从代码可以看出，这里就是做了一下简单的校验，然后便调用了`AMS.startService`启动`Service`，最终通过返回的`ComponentName`中的`packageName`来判断是否出错以及错误类型

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, boolean requireForeground, String callingPackage,
        String callingFeatureId, int userId)
        throws TransactionTooLargeException {
    enforceNotIsolatedCaller("startService");
    // Refuse possible leaked file descriptors
    //校验Intent，不允许其携带fd
    if (service != null && service.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    //调用方包名不能为空
    if (callingPackage == null) {
        throw new IllegalArgumentException("callingPackage cannot be null");
    }

    synchronized(this) {
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        ComponentName res;
        try {
            //调用ActiveServices.startServiceLocked方法
            res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid,
                    requireForeground, callingPackage, callingFeatureId, userId);
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
        return res;
    }
}
```

同样，这里做了一些简单的检查，然后调用`ActiveServices.startServiceLocked`方法，`ActiveServices`是一个辅助`AMS`进行`Service`管理的类，包括`Service`的启动、绑定和停止等

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired, String callingPackage,
        @Nullable String callingFeatureId, final int userId)
        throws TransactionTooLargeException {
    return startServiceLocked(caller, service, resolvedType, callingPid, callingUid, fgRequired,
            callingPackage, callingFeatureId, userId, false);
}

ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired, String callingPackage,
        @Nullable String callingFeatureId, final int userId,
        boolean allowBackgroundActivityStarts) throws TransactionTooLargeException {
    //判断调用方是否为前台
    final boolean callerFg;
    if (caller != null) {
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        if (callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + callingPid
                    + ") when starting service " + service);
        }
        callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
    } else {
        callerFg = true;
    }

    //查找待启动Service
    ServiceLookupResult res =
        retrieveServiceLocked(service, null, resolvedType, callingPackage,
                callingPid, callingUid, userId, true, callerFg, false, false);
    //如果找不到待启动Service，直接返回null
    if (res == null) {
        return null;
    }
    //如果待启动的Service所在package和uid无法与调用方package和uid建立关联，则无法启动Service
    //返回异常ComponentName，由上层抛出SecurityException异常
    if (res.record == null) {
        return new ComponentName("!", res.permission != null
                ? res.permission : "private to package");
    }

    ServiceRecord r = res.record;

    //试图用一个不存在的用户启动Service
    if (!mAm.mUserController.exists(r.userId)) {
        Slog.w(TAG, "Trying to start service with non-existent user! " + r.userId);
        return null;
    }

    // If we're starting indirectly (e.g. from PendingIntent), figure out whether
    // we're launching into an app in a background state.  This keys off of the same
    // idleness state tracking as e.g. O+ background service start policy.
    //Service所在应用未启动或处在后台
    final boolean bgLaunch = !mAm.isUidActiveLocked(r.appInfo.uid);

    // If the app has strict background restrictions, we treat any bg service
    // start analogously to the legacy-app forced-restrictions case, regardless
    // of its target SDK version.
    //检查Service所在应用后台启动限制
    boolean forcedStandby = false;
    if (bgLaunch && appRestrictedAnyInBackground(r.appInfo.uid, r.packageName)) {
        forcedStandby = true;
    }

    // If this is a direct-to-foreground start, make sure it is allowed as per the app op.
    boolean forceSilentAbort = false;
    if (fgRequired) { //作为前台服务启动
        //权限检查
        final int mode = mAm.getAppOpsManager().checkOpNoThrow(
                AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName);
        switch (mode) {
            //默认和允许都可以作为前台服务启动
            case AppOpsManager.MODE_ALLOWED:
            case AppOpsManager.MODE_DEFAULT:
                // All okay.
                break;
            //不允许的话，回退到作为普通后台服务启动
            case AppOpsManager.MODE_IGNORED:
                // Not allowed, fall back to normal start service, failing siliently
                // if background check restricts that.
                Slog.w(TAG, "startForegroundService not allowed due to app op: service "
                        + service + " to " + r.shortInstanceName
                        + " from pid=" + callingPid + " uid=" + callingUid
                        + " pkg=" + callingPackage);
                fgRequired = false;
                forceSilentAbort = true;
                break;
            //错误的话直接返回，由上层抛出SecurityException异常
            default:
                return new ComponentName("!!", "foreground not allowed as per app op");
        }
    }

    // If this isn't a direct-to-foreground start, check our ability to kick off an
    // arbitrary service
    //如果不是从前台启动
    //startRequested表示Service是否由startService方式所启动，fgRequired表示作为前台服务启动
    if (forcedStandby || (!r.startRequested && !fgRequired)) {
        // Before going further -- if this app is not allowed to start services in the
        // background, then at this point we aren't going to let it period.
        //服务是否允许在后台启动
        final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
        //如果不允许，则无法启动服务
        if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
            //静默的停止启动
            if (allowed == ActivityManager.APP_START_MODE_DELAYED || forceSilentAbort) {
                // In this case we are silently disabling the app, to disrupt as
                // little as possible existing apps.
                return null;
            }
            if (forcedStandby) {
                // This is an O+ app, but we might be here because the user has placed
                // it under strict background restrictions.  Don't punish the app if it's
                // trying to do the right thing but we're denying it for that reason.
                if (fgRequired) {
                    return null;
                }
            }
            // This app knows it is in the new model where this operation is not
            // allowed, so tell it what has happened.
            //明确的告知不允许启动，上层抛出异常
            UidRecord uidRec = mAm.mProcessList.getUidRecordLocked(r.appInfo.uid);
            return new ComponentName("?", "app is in background uid " + uidRec);
        }
    }

    // At this point we've applied allowed-to-start policy based on whether this was
    // an ordinary startService() or a startForegroundService().  Now, only require that
    // the app follow through on the startForegroundService() -> startForeground()
    // contract if it actually targets O+.
    //对于targetSdk 26以下（Android 8.0以下）的应用来说，不需要作为前台服务启动
    if (r.appInfo.targetSdkVersion < Build.VERSION_CODES.O && fgRequired) {
        fgRequired = false;
    }

    //检查通过Intent被临时授权的Uris
    NeededUriGrants neededGrants = mAm.mUgmInternal.checkGrantUriPermissionFromIntent(
            service, callingUid, r.packageName, r.userId);

    // If permissions need a review before any of the app components can run,
    // we do not start the service and launch a review activity if the calling app
    // is in the foreground passing it a pending intent to start the service when
    // review is completed.

    // XXX This is not dealing with fgRequired!
    //如果待启动的Service需要相应权限，则需要用户手动确认权限后，再进行启动
    if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage, callingFeatureId,
            callingUid, service, callerFg, userId)) {
        return null;
    }

    //取消之前的Service重启任务（如果有）
    if (unscheduleServiceRestartLocked(r, callingUid, false)) {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
    }
    r.lastActivity = SystemClock.uptimeMillis();
    //表示Service是否由startService方式所启动的
    r.startRequested = true;
    r.delayedStop = false;
    //是否作为前台服务启动
    r.fgRequired = fgRequired;
    //加入等待启动队列
    r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
            service, neededGrants, callingUid));

    //作为前台服务启动
    if (fgRequired) {
        // We are now effectively running a foreground service.
        ... //使用ServiceState记录
        ... //通过AppOpsService监控
    }

    final ServiceMap smap = getServiceMapLocked(r.userId);
    boolean addToStarting = false;
    //对于后台启动的非前台服务，需要判断其是否需要延迟启动
    if (!callerFg && !fgRequired && r.app == null
            && mAm.mUserController.hasStartedUserState(r.userId)) {
        //获取Service所处进程信息
        ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
        //没有对应进程（App未打开）或进程状态级别低于 [进程在后台运行Receiver]
        if (proc == null || proc.getCurProcState() > ActivityManager.PROCESS_STATE_RECEIVER) {
            // If this is not coming from a foreground caller, then we may want
            // to delay the start if there are already other background services
            // that are starting.  This is to avoid process start spam when lots
            // of applications are all handling things like connectivity broadcasts.
            // We only do this for cached processes, because otherwise an application
            // can have assumptions about calling startService() for a service to run
            // in its own process, and for that process to not be killed before the
            // service is started.  This is especially the case for receivers, which
            // may start a service in onReceive() to do some additional work and have
            // initialized some global state as part of that.
            //对于之前已经设置为延迟启动的服务，直接返回
            if (r.delayed) {
                // This service is already scheduled for a delayed start; just leave
                // it still waiting.
                return r.name;
            }
            //如果当前正在后台启动的Service数大于等于允许同时在后台启动的最大服务数
            //将这个Service设置为延迟启动
            if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                // Something else is starting, delay!
                smap.mDelayedStartList.add(r);
                r.delayed = true;
                return r.name;
            }
            //添加到正在启动服务列表中
            addToStarting = true;
        } else if (proc.getCurProcState() >= ActivityManager.PROCESS_STATE_SERVICE) {
            //进程状态为 [正在运行Service的后台进程] 或 [正在运行Receiver的后台进程] 时
            // We slightly loosen when we will enqueue this new service as a background
            // starting service we are waiting for, to also include processes that are
            // currently running other services or receivers.
            //添加到正在启动服务列表中
            addToStarting = true;
        }
    }

    //如果允许Service后台启动Activity，则将其加入到白名单中
    if (allowBackgroundActivityStarts) {
        r.whitelistBgActivityStartsOnServiceStart();
    }
    //继续启动Service
    ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);

    //检查是否允许前台服务使用while-in-use权限
    if (!r.mAllowWhileInUsePermissionInFgs) {
        r.mAllowWhileInUsePermissionInFgs =
                shouldAllowWhileInUsePermissionInFgsLocked(callingPackage, callingPid,
                        callingUid, service, r, allowBackgroundActivityStarts);
    }

    return cmp;
}
```
这个方法涉及到很多前后台判断，我想这里的前后台其实分为三个概念，一是调用方App是否在前台，二是`Service`方App是否在前台，三是`Service`是否作为前台服务启动，当然，大部分情况启动的都是App内的`Service`，即一二中的前后台状态是一致的，但也不排除启动其他App的`Service`这种情况，所以这里还是需要好好区分开来

这个方法看起来很长，但总之都是一些`Service`启动前的预处理工作，主要做了以下几点工作：

1. 判断调用方进程是否在前台（`callerFg`）：对于调用方在后台启动的`Service`，需要判断其是否需要延迟启动
2. 调用`retrieveServiceLocked`查找待启动`Service`信息（`ServiceRecord`）
3. 各种检查，一旦发现不满足启动条件就终止启动`Service`
4. 检查`Service`所在应用的前后台状态以及后台启动限制，不符合条件则终止启动`Service`
5. 判断是否可以作为前台服务启动
6. 如果待启动的`Service`需要相应权限，则需要用户手动确认权限后，再进行启动
7. 取消之前的`Service`重启任务（如果有）
8. 设置`ServiceRecord`状态，包括上次活动时间，是否由startService方式所启动的，是否作为前台服务启动等
9. 如果作为前台服务启动，则需要进行记录和监控
10. 对于后台启动的非前台服务，需要判断其是否需要延迟启动
11. 调用`startServiceInnerLocked`继续启动`Service`

```java
//frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
        boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
    ... //记录
    //启动前初始化
    r.callStart = false;
    ... //记录
    //拉起服务，如果服务未启动，则会启动服务并调用其onCreate和onStartCommond方法
    //如果服务已启动则会直接调用其onStartCommond方法
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
    if (error != null) {
        return new ComponentName("!!", error);
    }

    if (r.startRequested && addToStarting) { //对于后台启动服务的情况
        //是否为第一个后台启动的服务
        boolean first = smap.mStartingBackground.size() == 0;
        //添加到正在后台启动服务列表中
        smap.mStartingBackground.add(r);
        //设置后台启动服务超时时间（默认15秒）
        r.startingBgTimeout = SystemClock.uptimeMillis() + mAm.mConstants.BG_START_TIMEOUT;
        //如果为第一个后台启动的服务，则代表后面暂时没有正在后台启动的服务了
        //此时将之前设置为延迟启动的服务调度出来后台启动
        if (first) {
            smap.rescheduleDelayedStartsLocked();
        }
    } else if (callerFg || r.fgRequired) { //对于调用方进程为前台或作为前台服务启动的情况
        //将此Service从正在后台启动服务列表和延迟启动服务列表中移除
        //如果正在后台启动服务列表中存在此服务的话，将之前设置为延迟启动的服务调度出来后台启动
        smap.ensureNotStartingBackgroundLocked(r);
    }

    return r.name;
}
```

```java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting, boolean permissionsReviewRequired)
        throws TransactionTooLargeException {
    if (r.app != null && r.app.thread != null) {
        sendServiceArgsLocked(r, execInFg, false);
        return null;
    }

    if (!whileRestarting && mRestartingServices.contains(r)) {
        // If waiting for a restart, then do nothing.
        return null;
    }

    // We are now bringing the service up, so no longer in the
    // restarting state.
    if (mRestartingServices.remove(r)) {
        clearRestartingIfNeededLocked(r);
    }

    // Make sure this service is no longer considered delayed, we are starting it now.
    if (r.delayed) {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
        getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
        r.delayed = false;
    }

    // Make sure that the user who owns this service is started.  If not,
    // we don't want to allow it to run.
    if (!mAm.mUserController.hasStartedUserState(r.userId)) {
        String msg = "Unable to launch app "
                + r.appInfo.packageName + "/"
                + r.appInfo.uid + " for service "
                + r.intent.getIntent() + ": user " + r.userId + " is stopped";
        Slog.w(TAG, msg);
        bringDownServiceLocked(r);
        return msg;
    }

    // Service is now being launched, its package can't be stopped.
    try {
        AppGlobals.getPackageManager().setPackageStoppedState(
                r.packageName, false, r.userId);
    } catch (RemoteException e) {
    } catch (IllegalArgumentException e) {
        Slog.w(TAG, "Failed trying to unstop package "
                + r.packageName + ": " + e);
    }

    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
    final String procName = r.processName;
    HostingRecord hostingRecord = new HostingRecord("service", r.instanceName);
    ProcessRecord app;

    if (!isolated) {
        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
        if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                    + " app=" + app);
        if (app != null && app.thread != null) {
            try {
                app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                realStartServiceLocked(r, app, execInFg);
                return null;
            } catch (TransactionTooLargeException e) {
                throw e;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting service " + r.shortInstanceName, e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }
    } else {
        // If this service runs in an isolated process, then each time
        // we call startProcessLocked() we will get a new isolated
        // process, starting another process if we are currently waiting
        // for a previous process to come up.  To deal with this, we store
        // in the service any current isolated process it is running in or
        // waiting to have come up.
        app = r.isolatedProc;
        if (WebViewZygote.isMultiprocessEnabled()
                && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
            hostingRecord = HostingRecord.byWebviewZygote(r.instanceName);
        }
        if ((r.serviceInfo.flags & ServiceInfo.FLAG_USE_APP_ZYGOTE) != 0) {
            hostingRecord = HostingRecord.byAppZygote(r.instanceName, r.definingPackageName,
                    r.definingUid);
        }
    }

    // Not running -- get it started, and enqueue this service record
    // to be executed when the app comes up.
    if (app == null && !permissionsReviewRequired) {
        // TODO (chriswailes): Change the Zygote policy flags based on if the launch-for-service
        //  was initiated from a notification tap or not.
        if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, false, isolated, false)) == null) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": process is bad";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }
        if (isolated) {
            r.isolatedProc = app;
        }
    }

    if (r.fgRequired) {
        if (DEBUG_FOREGROUND_SERVICE) {
            Slog.v(TAG, "Whitelisting " + UserHandle.formatUid(r.appInfo.uid)
                    + " for fg-service launch");
        }
        mAm.tempWhitelistUidLocked(r.appInfo.uid,
                SERVICE_START_FOREGROUND_TIMEOUT, "fg-service-launch");
    }

    if (!mPendingServices.contains(r)) {
        mPendingServices.add(r);
    }

    if (r.delayedStop) {
        // Oh and hey we've already been asked to stop!
        r.delayedStop = false;
        if (r.startRequested) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                    "Applying delayed stop (in bring up): " + r);
            stopServiceLocked(r);
        }
    }

    return null;
}
```