---
title: Android源码分析 - Activity启动流程（上）
date: 2022-12-19 21:49:36
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

上一篇文章 [Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128) 中，我们分析了`App`进程的启动过程，包括`Application`是怎么创建并执行`onCreate`方法的，本篇文章我们将会继续分析`App`进程启动、`Application`创建完成后，`Activity`是如何启动的

# 两种路径启动Activity

我们在 [Android源码分析 - Activity启动流程（上）](https://juejin.cn/post/7130182223231188999) 的末尾分析过，`Activity`的启动存在两条路径

## 启动进程，然后启动Activity

这条路径就是我们在上一篇文章 [Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128) 中分析了前半部分：启动进程

当进程启动后，会执行到`AMS.attachApplicationLocked`方法，在这个方法的最后，会有一段代码检查是否有`Activity`等待启动

```java
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
        int pid, int callingUid, long startSeq) {
    ...

    // See if the top visible activity is waiting to run in this process...
    //检查是否有Activity等待启动
    if (normalMode) {
        try {
            didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
        } catch (Exception e) {
            badApp = true;
        }
    }

    ...

    return true;
}
```

然后调用`ActivityTaskManagerInternal.attachApplication`，`ActivityTaskManagerInternal`是一个抽象类，被`ATMS`的内部类`LocalService`实现

```java
public boolean attachApplication(WindowProcessController wpc) throws RemoteException {
    synchronized (mGlobalLockWithoutBoost) {
        try {
            return mRootWindowContainer.attachApplication(wpc);
        } finally {
            ...
        }
    }
}
```

接着调用`RootWindowContainer.attachApplication`

```java
boolean attachApplication(WindowProcessController app) throws RemoteException {
    boolean didSomething = false;
    for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
        mTmpRemoteException = null;
        mTmpBoolean = false; // Set to true if an activity was started.

        final DisplayContent display = getChildAt(displayNdx);
        for (int areaNdx = display.getTaskDisplayAreaCount() - 1; areaNdx >= 0; --areaNdx) {
            final TaskDisplayArea taskDisplayArea = display.getTaskDisplayAreaAt(areaNdx);
            for (int taskNdx = taskDisplayArea.getStackCount() - 1; taskNdx >= 0; --taskNdx) {
                final ActivityStack rootTask = taskDisplayArea.getStackAt(taskNdx);
                if (rootTask.getVisibility(null /*starting*/) == STACK_VISIBILITY_INVISIBLE) {
                    break;
                }
                //遍历ActivityStack下的所有ActivityRecord，
                //以其为参数调用startActivityForAttachedApplicationIfNeeded方法
                final PooledFunction c = PooledLambda.obtainFunction(
                        RootWindowContainer::startActivityForAttachedApplicationIfNeeded, this,
                        PooledLambda.__(ActivityRecord.class), app,
                        rootTask.topRunningActivity());
                rootTask.forAllActivities(c);
                c.recycle();
                if (mTmpRemoteException != null) {
                    throw mTmpRemoteException;
                }
            }
        }
        didSomething |= mTmpBoolean;
    }
    if (!didSomething) {
        ensureActivitiesVisible(null, 0, false /* preserve_windows */);
    }
    return didSomething;
}
```

这里有两层`for`循环，可以看出来，这个方法遍历了所有可见的`ActivityStack`，然后再对每个可见的`ActivityStack`进行操作

其中，`PooledLambda`这个类采用了池化技术，用于构造可回收复用的匿名函数，这里`PooledLambda.obtainFunction`方法得到的结果是一个`Function<ActivityRecord, Boolean>`，`forAllActivities`方法被定义在父类`WindowContainer`中，就是遍历执行所有`child`的`forAllActivities`方法，而`ActivityRecord`中重写了这个方法，直接用自身执行这个`Function`

简单来说，可以将这部分代码看作以下伪代码：

```java
rootTask.forEachAllActivityRecord((activityRecord) -> {
    startActivityForAttachedApplicationIfNeeded(activityRecord, app, rootTask.topRunningActivity())
})
```

接着我们来观察`startActivityForAttachedApplicationIfNeeded`方法

```java
private boolean startActivityForAttachedApplicationIfNeeded(ActivityRecord r,
        WindowProcessController app, ActivityRecord top) {
    //Activity是否正在finish，是否应为当前用户显示Activity，忽略锁屏情况，此Activity是否可见
    //对比Activity与发起进程的uid和进程名
    if (r.finishing || !r.okToShowLocked() || !r.visibleIgnoringKeyguard || r.app != null
            || app.mUid != r.info.applicationInfo.uid || !app.mName.equals(r.processName)) {
        return false;
    }

    try {
        //启动Activity
        if (mStackSupervisor.realStartActivityLocked(r, app,
                top == r && r.isFocusable() /*andResume*/, true /*checkConfig*/)) {
            mTmpBoolean = true;
        }
    } catch (RemoteException e) {
        ...
        return true;
    }
    return false;
}
```

上面一系列的判断检查传入的`ActivityRecord`所对应的`Activity`是否需要启动，然后调用`ActivityStackSupervisor.realStartActivityLocked`方法启动`Activity`

## 已有进程，直接启动Activity

这条路径我在 [Android源码分析 - Activity启动流程（上）](https://juejin.cn/post/7130182223231188999) 中的末尾也分析过，如果`App`进程已经启动，则会调用`ActivityStackSupervisor.startSpecificActivity`方法

```java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
    if (wpc != null && wpc.hasThread()) {
        try {
            //启动Activity
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }

    //锁屏状态下启动Activity防闪烁机制
    r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

    //出现异常，重启进程
    final boolean isTop = andResume && r.isTopRunningActivity();
    mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
}
```

可以看到，这里也调用了`ActivityStackSupervisor.realStartActivityLocked`方法启动`Activity`

# realStartActivityLocked

最终两条路径都走到了`ActivityStackSupervisor.realStartActivityLocked`方法中，那我们就来看看这个方法做了什么

```java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
        boolean andResume, boolean checkConfig) throws RemoteException {

    //确保所有Activity都已暂停
    if (!mRootWindowContainer.allPausedActivitiesComplete()) {
        // While there are activities pausing we skipping starting any new activities until
        // pauses are complete. NOTE: that we also do this for activities that are starting in
        // the paused state because they will first be resumed then paused on the client side.
        return false;
    }

    final Task task = r.getTask();
    final ActivityStack stack = task.getStack();

    //延迟resume以避免重复resume
    //通过设置标记位mDeferResumeCount，只有当其为0时才能执行resume
    beginDeferResume();

    try {
        //冻结屏幕（不接收输入、不执行动画，截取屏幕进行显示）
        r.startFreezingScreenLocked(proc, 0);

        // schedule launch ticks to collect information about slow apps.
        //收集启动缓慢信息
        r.startLaunchTickingLocked();

        r.setProcess(proc);

        // Ensure activity is allowed to be resumed after process has set.
        //确保Activity允许被resume
        if (andResume && !r.canResumeByCompat()) {
            andResume = false;
        }

        //锁屏状态下启动Activity防闪烁机制
        r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

        // Have the window manager re-evaluate the orientation of the screen based on the new
        // activity order.  Note that as a result of this, it can call back into the activity
        // manager with a new orientation.  We don't care about that, because the activity is
        // not currently running so we are just restarting it anyway.
        if (checkConfig) {
            // Deferring resume here because we're going to launch new activity shortly.
            // We don't want to perform a redundant launch of the same record while ensuring
            // configurations and trying to resume top activity of focused stack.
            //确保所有Activity的可见性、更新屏幕方向和配置
            mRootWindowContainer.ensureVisibilityAndConfig(r, r.getDisplayId(),
                    false /* markFrozenIfConfigChanged */, true /* deferResume */);
        }

        //检查Activity是否在后台锁屏状态下启动
        if (r.getRootTask().checkKeyguardVisibility(r, true /* shouldBeVisible */,
                true /* isTop */) && r.allowMoveToFront()) {
            // We only set the visibility to true if the activity is not being launched in
            // background, and is allowed to be visible based on keyguard state. This avoids
            // setting this into motion in window manager that is later cancelled due to later
            // calls to ensure visible activities that set visibility back to false.
            //只有Activity不是在后台启动，才将其可见性设置为true
            r.setVisibility(true);
        }

        ... //异常情况检查

        r.launchCount++;
        r.lastLaunchTime = SystemClock.uptimeMillis();
        proc.setLastActivityLaunchTime(r.lastLaunchTime);

        //屏幕固定功能
        final LockTaskController lockTaskController = mService.getLockTaskController();
        if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE
                || task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV
                || (task.mLockTaskAuth == LOCK_TASK_AUTH_WHITELISTED
                        && lockTaskController.getLockTaskModeState()
                                == LOCK_TASK_MODE_LOCKED)) {
            lockTaskController.startLockTaskMode(task, false, 0 /* blank UID */);
        }

        try {
            if (!proc.hasThread()) {
                throw new RemoteException();
            }
            List<ResultInfo> results = null;
            List<ReferrerIntent> newIntents = null;
            if (andResume) {
                // We don't need to deliver new intents and/or set results if activity is going
                // to pause immediately after launch.
                results = r.results;
                newIntents = r.newIntents;
            }
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                    "Launching: " + r + " savedState=" + r.getSavedState()
                            + " with results=" + results + " newIntents=" + newIntents
                            + " andResume=" + andResume);
            EventLogTags.writeWmRestartActivity(r.mUserId, System.identityHashCode(r),
                    task.mTaskId, r.shortComponentName);
            if (r.isActivityTypeHome()) {
                // Home process is the root process of the task.
                updateHomeProcess(task.getBottomMostActivity().app);
            }
            mService.getPackageManagerInternalLocked().notifyPackageUse(
                    r.intent.getComponent().getPackageName(), NOTIFY_PACKAGE_USE_ACTIVITY);
            r.setSleeping(false);
            r.forceNewConfig = false;
            mService.getAppWarningsLocked().onStartActivity(r);
            r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);

            // Because we could be starting an Activity in the system process this may not go
            // across a Binder interface which would create a new Configuration. Consequently
            // we have to always create a new Configuration here.

            final MergedConfiguration mergedConfiguration = new MergedConfiguration(
                    proc.getConfiguration(), r.getMergedOverrideConfiguration());
            r.setLastReportedConfiguration(mergedConfiguration);

            logIfTransactionTooLarge(r.intent, r.getSavedState());


            // Create activity launch transaction.
            final ClientTransaction clientTransaction = ClientTransaction.obtain(
                    proc.getThread(), r.appToken);

            final DisplayContent dc = r.getDisplay().mDisplayContent;
            clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                    System.identityHashCode(r), r.info,
                    // TODO: Have this take the merged configuration instead of separate global
                    // and override configs.
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                    r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                    dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                    r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));

            // Set desired final state.
            final ActivityLifecycleItem lifecycleItem;
            if (andResume) {
                lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
            } else {
                lifecycleItem = PauseActivityItem.obtain();
            }
            clientTransaction.setLifecycleStateRequest(lifecycleItem);

            // Schedule transaction.
            mService.getLifecycleManager().scheduleTransaction(clientTransaction);

            if ((proc.mInfo.privateFlags & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0
                    && mService.mHasHeavyWeightFeature) {
                // This may be a heavy-weight process! Note that the package manager will ensure
                // that only activity can run in the main process of the .apk, which is the only
                // thing that will be considered heavy-weight.
                if (proc.mName.equals(proc.mInfo.packageName)) {
                    if (mService.mHeavyWeightProcess != null
                            && mService.mHeavyWeightProcess != proc) {
                        Slog.w(TAG, "Starting new heavy weight process " + proc
                                + " when already running "
                                + mService.mHeavyWeightProcess);
                    }
                    mService.setHeavyWeightProcess(r);
                }
            }

        } catch (RemoteException e) {
            if (r.launchFailed) {
                // This is the second time we failed -- finish activity and give up.
                Slog.e(TAG, "Second failure launching "
                        + r.intent.getComponent().flattenToShortString() + ", giving up", e);
                proc.appDied("2nd-crash");
                r.finishIfPossible("2nd-crash", false /* oomAdj */);
                return false;
            }

            // This is the first time we failed -- restart process and
            // retry.
            r.launchFailed = true;
            proc.removeActivity(r);
            throw e;
        }
    } finally {
        endDeferResume();
    }

    r.launchFailed = false;

    // TODO(lifecycler): Resume or pause requests are done as part of launch transaction,
    // so updating the state should be done accordingly.
    if (andResume && readyToResume()) {
        // As part of the process of launching, ActivityThread also performs
        // a resume.
        stack.minimalResumeActivityLocked(r);
    } else {
        // This activity is not starting in the resumed state... which should look like we asked
        // it to pause+stop (but remain visible), and it has done so and reported back the
        // current icicle and other state.
        if (DEBUG_STATES) Slog.v(TAG_STATES,
                "Moving to PAUSED: " + r + " (starting in paused state)");
        r.setState(PAUSED, "realStartActivityLocked");
        mRootWindowContainer.executeAppTransitionForAllDisplay();
    }
    // Perform OOM scoring after the activity state is set, so the process can be updated with
    // the latest state.
    proc.onStartActivity(mService.mTopProcessState, r.info);

    // Launch the new version setup screen if needed.  We do this -after-
    // launching the initial activity (that is, home), so that it can have
    // a chance to initialize itself while in the background, making the
    // switch back to it faster and look better.
    if (mRootWindowContainer.isTopDisplayFocusedStack(stack)) {
        mService.getActivityStartController().startSetupActivity();
    }

    // Update any services we are bound to that might care about whether
    // their client may have activities.
    if (r.app != null) {
        r.app.updateServiceConnectionActivities();
    }

    return true;
}
```