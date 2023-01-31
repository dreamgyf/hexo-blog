---
title: Android源码分析 - Activity启动流程（下）
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
            //如果是ACTIVITY_TYPE_HOME类型的应用（Launcher）
            if (r.isActivityTypeHome()) {
                // Home process is the root process of the task.
                updateHomeProcess(task.getBottomMostActivity().app);
            }
            //信息记录
            mService.getPackageManagerInternalLocked().notifyPackageUse(
                    r.intent.getComponent().getPackageName(), NOTIFY_PACKAGE_USE_ACTIVITY);
            r.setSleeping(false);
            r.forceNewConfig = false;
            //如果有必要的话，显示一些App警告弹窗（不支持的CompileSdk、不支持的TargetSdk、不支持的显示大小）
            mService.getAppWarningsLocked().onStartActivity(r);
            //兼容性信息
            r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);

            // Because we could be starting an Activity in the system process this may not go
            // across a Binder interface which would create a new Configuration. Consequently
            // we have to always create a new Configuration here.

            final MergedConfiguration mergedConfiguration = new MergedConfiguration(
                    proc.getConfiguration(), r.getMergedOverrideConfiguration());
            r.setLastReportedConfiguration(mergedConfiguration);


            // Create activity launch transaction.
            //创建或获取一个client事务
            final ClientTransaction clientTransaction = ClientTransaction.obtain(
                    proc.getThread(), r.appToken);

            final DisplayContent dc = r.getDisplay().mDisplayContent;
            //添加一条Activity启动消息
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
            //设置client执行事务后应处于的生命周期状态
            clientTransaction.setLifecycleStateRequest(lifecycleItem);

            // Schedule transaction.
            //调度执行此事务，启动Activity
            mService.getLifecycleManager().scheduleTransaction(clientTransaction);

            //处理重量级进程
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
                //第二次启动失败，finish掉Activity并放弃重试，直接返回false
                Slog.e(TAG, "Second failure launching "
                        + r.intent.getComponent().flattenToShortString() + ", giving up", e);
                proc.appDied("2nd-crash");
                r.finishIfPossible("2nd-crash", false /* oomAdj */);
                return false;
            }

            // This is the first time we failed -- restart process and
            // retry.
            //第一次启动失败，尝试重启进程并重试启动Activity
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
    //更新生命周期状态
    if (andResume && readyToResume()) {
        // As part of the process of launching, ActivityThread also performs
        // a resume.
        stack.minimalResumeActivityLocked(r);
    } else {
        // This activity is not starting in the resumed state... which should look like we asked
        // it to pause+stop (but remain visible), and it has done so and reported back the
        // current icicle and other state.
        r.setState(PAUSED, "realStartActivityLocked");
        mRootWindowContainer.executeAppTransitionForAllDisplay();
    }
    // Perform OOM scoring after the activity state is set, so the process can be updated with
    // the latest state.
    //更新进程oom adj，更新进程状态
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
    //更新进程绑定的所有服务
    if (r.app != null) {
        r.app.updateServiceConnectionActivities();
    }

    return true;
}
```

这个方法中最关键的部分在于创建了`ClientTransaction`事务，并向里添加了一条启动`Activity`的消息，然后调用`ATMS.getLifecycleManager.scheduleTransaction`调度执行这个事务，启动`Activity`

# ClientTransaction

我们先来看看`ClientTransaction`这个对象是怎么创建获取的，我们首先调用了`ClientTransaction.obtain`方法，并传入了一个`ActivityThread`内部类`ApplicationThread`的`Binder`对象`IApplicationThread`和一个`ActivityRecord.Token`对象

```java
public static ClientTransaction obtain(IApplicationThread client, IBinder activityToken) {
    ClientTransaction instance = ObjectPool.obtain(ClientTransaction.class);
    if (instance == null) {
        instance = new ClientTransaction();
    }
    instance.mClient = client;
    instance.mActivityToken = activityToken;

    return instance;
}
```

这个方法很简单，从池子里拿一个实例，或者新创建一个实例对象，将这个实例对象的两个成员变量赋值后返回

然后我们调用了`ClientTransaction.addCallback`方法将`ClientTransactionItem`加入到回调队列中，`ClientTransactionItem`是一条能够被执行的生命周期回调消息，它是一个抽象类，子类需要实现它的`preExecute`、`execute`、`postExecute`方法

我们这里传入的是`LaunchActivityItem`，这条消息是用来启动`Activity`的

然后我们调用`ClientTransaction.setLifecycleStateRequest`设置当事务执行结束后，`Activity`应该处在一个怎样的生命周期

最后调用`ATMS.getLifecycleManager.scheduleTransaction`调度执行这个事务，`ATMS.getLifecycleManager`获得的是一个`ClientLifecycleManager`对象，我们沿着这个方法继续往下看

```java
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    transaction.schedule();
    if (!(client instanceof Binder)) {
        // If client is not an instance of Binder - it's a remote call and at this point it is
        // safe to recycle the object. All objects used for local calls will be recycled after
        // the transaction is executed on client in ActivityThread.
        transaction.recycle();
    }
}
```

除了回收调用`recycle`之外，我们又回到了`ClientTransaction`中，调用其`schedule`方法

```java
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```

这里又跨进程回到了`App`进程中，调用`ApplicationThread.scheduleTransaction`

```java
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}
```

最后调用`ActivityThread.scheduleTransaction`执行事务，`ActivityThread`继承自`ClientTransactionHandler`，`scheduleTransaction`方法是在这里面定义的

```java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

## preExecute

首先，调用`ClientTransaction.preExecute`方法，然后通过`Handler`发送执行一条`EXECUTE_TRANSACTION`消息，我们先看一下`preExecute`

```java
public void preExecute(android.app.ClientTransactionHandler clientTransactionHandler) {
    if (mActivityCallbacks != null) {
        final int size = mActivityCallbacks.size();
        for (int i = 0; i < size; ++i) {
            mActivityCallbacks.get(i).preExecute(clientTransactionHandler, mActivityToken);
        }
    }
    if (mLifecycleStateRequest != null) {
        mLifecycleStateRequest.preExecute(clientTransactionHandler, mActivityToken);
    }
}
```

可以看到，就是遍历整个`callback`列表，执行`preExecute`方法，最后再执行`LifecycleStateRequest`的`preExecute`方法，对应到`Activity`启动流程中，就是先执行`LaunchActivityItem.preExecute`，再执行`ResumeActivityItem.preExecute`

```java
// LaunchActivityItem
public void preExecute(ClientTransactionHandler client, IBinder token) {
    client.countLaunchingActivities(1);
    client.updateProcessState(mProcState, false);
    client.updatePendingConfiguration(mCurConfig);
}
```
```java
// ResumeActivityItem
public void preExecute(ClientTransactionHandler client, IBinder token) {
    if (mUpdateProcState) {
        client.updateProcessState(mProcState, false);
    }
}
```

都是一些状态更新之类的东西，我们就直接跳过，然后我们看`ActivityThread`在收到`EXECUTE_TRANSACTION`消息后做了什么

```java
public void handleMessage(Message msg) {
    switch (msg.what) {
        ...
        case EXECUTE_TRANSACTION:
            final ClientTransaction transaction = (ClientTransaction) msg.obj;
            mTransactionExecutor.execute(transaction);
            ...
            break;
        ...
    }
    ...
}
```

## execute

这里可以看到，调用了`TransactionExecutor`对象的`execute`方法，`TransactionExecutor`对象在`ActivityThread`创建时便创建了，内部持有一个`ClientTransactionHandler`引用，即`ActivityThread`自身

```java
public void execute(ClientTransaction transaction) {
    final IBinder token = transaction.getActivityToken();
    //处理需要销毁的Activities
    if (token != null) {
        final Map<IBinder, ClientTransactionItem> activitiesToBeDestroyed =
                mTransactionHandler.getActivitiesToBeDestroyed();
        final ClientTransactionItem destroyItem = activitiesToBeDestroyed.get(token);
        if (destroyItem != null) {
            if (transaction.getLifecycleStateRequest() == destroyItem) {
                // It is going to execute the transaction that will destroy activity with the
                // token, so the corresponding to-be-destroyed record can be removed.
                activitiesToBeDestroyed.remove(token);
            }
            if (mTransactionHandler.getActivityClient(token) == null) {
                // The activity has not been created but has been requested to destroy, so all
                // transactions for the token are just like being cancelled.
                //Activity尚未被创建就被请求destroy，直接取消整个事务
                Slog.w(TAG, tId(transaction) + "Skip pre-destroyed transaction:\n"
                        + transactionToString(transaction, mTransactionHandler));
                return;
            }
        }
    }

    executeCallbacks(transaction);

    executeLifecycleState(transaction);
    mPendingActions.clear();
}
```

处理需要销毁的`Activities`这里我们不关注，就直接跳过，然后就是分别执行各个`ClientTransactionItem`回调消息，最后让其调度执行我们设置的最终的生命周期

```java
public void executeCallbacks(ClientTransaction transaction) {
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
    if (callbacks == null || callbacks.isEmpty()) {
        // No callbacks to execute, return early.
        return;
    }

    final IBinder token = transaction.getActivityToken();
    ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

    ... //生命周期转换相关

    final int size = callbacks.size();
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        ... //生命周期转换，在Activity启动时不会走进这个case

        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
        if (r == null) {
            // Launch activity request will create an activity record.
            //在执行完启动Activity后会新建一个ActivityClientRecord，重新赋值
            r = mTransactionHandler.getActivityClient(token);
        }

        ... //生命周期转换，在Activity启动时不会走进这个case
    }
}
```

关于生命周期转换，由于`Activity`启动的当前阶段不会进入这些case，所以等之后碰到了再细说

经过简化，实际上也就执行了`LaunchActivityItem.execute`和`LaunchActivityItem.postExecute`方法

```java
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
}
```

这里使用之前在`AMS`中创建`LaunchActivityItem`所使用到的信息，创建了一个`ActivityClientRecord`对象，接着回到`ActivityThread`，调用其`handleLaunchActivity`方法

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
    }

    // Make sure we are running with the most recent config.
    //确保Configuration为最新
    handleConfigurationChanged(null, null);

    // Initialize before creating the activity
    //初始化硬件加速
    if (!ThreadedRenderer.sRendererDisabled
            && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
        HardwareRenderer.preload();
    }
    //确保WMS被初始化
    WindowManagerGlobal.initialize();

    // Hint the GraphicsEnvironment that an activity is launching on the process.
    //通知有Activity启动
    GraphicsEnvironment.hintActivityLaunch();

    //执行启动Activity
    final Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        //设置Configuration
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        if (!r.activity.mFinished && pendingActions != null) {
            pendingActions.setOldState(r.state);
            pendingActions.setRestoreInstanceState(true);
            pendingActions.setCallOnPostCreate(true);
        }
    } else {
        // If there was an error, for any reason, tell the activity manager to stop us.
        //出现错误，停止启动Activity
        try {
            ActivityTaskManager.getService()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                            Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }

    return a;
}
```

这个方法中，最重要的莫过于`performLaunchActivity`了，它是创建`Activity`的核心方法

### performLaunchActivity

```java
/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ActivityInfo aInfo = r.activityInfo;
    //设置LoadedApk
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(
            mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    //如果启动的Activity是一个activity-alias，将Component设置为真正的Activity组件
    //详见：https://developer.android.com/guide/topics/manifest/activity-alias-element?hl=zh-cn
    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    //为Activity创建BaseContext
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //实例化Activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        ...
    }

    try {
        //创建或获取Application
        //如果该Activity指定在其他的一个新的进程中启动（设置了android:process属性），则会新创建Application
        //正常不涉及多进程，都是直接获取之前创建好的Application
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (activity != null) {
            //Manifest中Activity标签下的label属性
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            //准备Configuration
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            Window window = null;
            //当relaunch Activity的时候mPreserveWindow才会为true（比如说调用Activity.recreate方法）
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }

            // Activity resources must be initialized with the same loaders as the
            // application context.
            //设置Activity Resource的Loaders与Application Resource的Loaders一致
            appContext.getResources().addLoaders(
                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

            appContext.setOuterContext(activity);
            //重要：绑定BaseContext、创建PhoneWindow等一系列初始化工作
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            //更新网络状态
            checkAndBlockForNetworkAccess();
            activity.mStartedActivity = false;
            //设置主题
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            //调用Activity的onCreate方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                //在执行完super.onCreate方法后，mCalled会被置为true
                //如果mCalled为false，说明没有执行super.onCreate方法
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
            mLastReportedWindowingMode.put(activity.getActivityToken(),
                    config.windowConfiguration.getWindowingMode());
        }
        //设置生命周期状态为onCreate
        r.setState(ON_CREATE);

        // updatePendingActivityConfiguration() reads from mActivities to update
        // ActivityClientRecord which runs in a different thread. Protect modifications to
        // mActivities to avoid race.
        //将新建的ActivityClientRecord添加到mActivities中
        synchronized (mResourcesManager) {
            mActivities.put(r.token, r);
        }

    } catch (SuperNotCalledException e) {
        throw e;

    } catch (Exception e) {
        ...
    }

    return activity;
}
```

这个方法主要做了以下几个工作：

1. 准备创建`Activity`所必要的信息，譬如类名等

2. 为`Activity`创建`BaseContext`

3. 通过`Instrumentation`实例化`Activity`

4. 创建或获取`Activity`进程所对应的`Application`

5. 初始化`Activity`，执行各种绑定工作，创建`PhoneWindow`等

6. 执行`Activity`的`onCreate`生命周期方法

7. 将`ActivityClientRecord`生命周期状态设置为`onCreate`

我们重点看一下最主要的实例化、`attach`和`onCreate`这三点

#### Instrumentation.newActivity

我们在 [Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128/#heading-14) 中分析了，`Application`是怎么通过`Instrumentation`创建的，`Activity`的创建和它类似

```java
public Activity newActivity(ClassLoader cl, String className,
        Intent intent)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    String pkg = intent != null && intent.getComponent() != null
            ? intent.getComponent().getPackageName() : null;
    return getFactory(pkg).instantiateActivity(cl, className, intent);
}
```

同样的使用了`AppComponentFactory`创建，我们还是去看一下它的默认实现

```java
public @NonNull Activity instantiateActivity(@NonNull ClassLoader cl, @NonNull String className,
        @Nullable Intent intent)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Activity) cl.loadClass(className).newInstance();
}
```

同样的，也是通过类名反射创建一个`Activity`的实例

#### Activity.attach

紧接着，我们来看`Activity.attach`方法

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    //绑定BaseContext
    attachBaseContext(context);
    //初始化Fragment控制器
    mFragments.attachHost(null /*parent*/);

    //创建并设置Window用于显示界面
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(mWindowControllerCallback);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }

    //各成员变量初始化
    mUiThread = Thread.currentThread();

    mMainThread = aThread;
    mInstrumentation = instr;
    mToken = token;
    mAssistToken = assistToken;
    mIdent = ident;
    mApplication = application;
    mIntent = intent;
    mReferrer = referrer;
    mComponent = intent.getComponent();
    mActivityInfo = info;
    mTitle = title;
    mParent = parent;
    mEmbeddedID = id;
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
    if (voiceInteractor != null) {
        if (lastNonConfigurationInstances != null) {
            mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
        } else {
            mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                    Looper.myLooper());
        }
    }

    //设置WindowManager、ActivityRecordToken以及是否使用硬件加速
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;

    mWindow.setColorMode(info.colorMode);
    mWindow.setPreferMinimalPostProcessing(
            (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

    //设置自动填充选项
    setAutofillOptions(application.getAutofillOptions());
    //设置内容捕获功能
    setContentCaptureOptions(application.getContentCaptureOptions());
}
```

这里可以看到，`attach`方法主要做了以下几件事：

1. 绑定`BaseContext`

2. 初始化`Fragment`控制器

3. 创建并设置`Window`

4. 各种成员变量及其他属性初始化

看完这个方法，我们可以发现，原来`Activity`的`Window`是在这个时候创建的，并且`Window`的具体实现类为`PhoneWindow`

#### Instrumentation.callActivityOnCreate

再然后便是通过`Instrumentation`执行`Activity`的`onCreate`生命周期方法了

```java
public void callActivityOnCreate(Activity activity, Bundle icicle) {
    prePerformCreate(activity);
    activity.performCreate(icicle);
    postPerformCreate(activity);
}
```

其中`prePerformCreate`和`postPerformCreate`似乎只有在单元测试和CTS测试下才会产生实质性的影响，在正常情况下我们就当作它们不存在，我们接着看`performCreate`方法

```java
final void performCreate(Bundle icicle) {
    performCreate(icicle, null);
}

final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    //分发PreCreated事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPreCreated回调
    dispatchActivityPreCreated(icicle);
    mCanEnterPictureInPicture = true;
    // initialize mIsInMultiWindowMode and mIsInPictureInPictureMode before onCreate
    final int windowingMode = getResources().getConfiguration().windowConfiguration
            .getWindowingMode();
    mIsInMultiWindowMode = inMultiWindowMode(windowingMode);
    mIsInPictureInPictureMode = windowingMode == WINDOWING_MODE_PINNED;
    restoreHasCurrentPermissionRequest(icicle);
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
    mActivityTransitionState.readState(icicle);

    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
            com.android.internal.R.styleable.Window_windowNoDisplay, false);
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    //分发PostCreated事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPostCreated回调
    dispatchActivityPostCreated(icicle);
}
```

其中的参数`icicle`就是我们平时重写`Activity.onCreate`方法时的第一个入参`savedInstanceState`，如果`Activity`发生了重建之类的情况，它会保存一些状态数据，第一次启动`Activity`时为`null`

