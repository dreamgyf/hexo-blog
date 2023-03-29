---
title: Android源码分析 - Activity销毁流程
date: 2023-03-08 15:49:50
tags: 
- Android源码
- ActivityThread
- ActivityManagerService
- ActivityTaskManagerService
categories: 
- [Android, 源码分析]
- [Android, ActivityThread]
- [Android, ActivityManagerService]
- [Android, ActivityTaskManagerService]
---

# 开篇

**本篇以android-11.0.0_r25作为基础解析**

我们在之前的几篇`Activity`启动流程分析中已经了解了`Activity`一半的生命周期，接下来这篇文章我们就来分析一下`Activity`销毁相关的生命周期

前几期文章回顾：

[Android源码分析 - Activity启动流程（上）](https://juejin.cn/post/7130182223231188999)

[Android源码分析 - Activity启动流程（中）](https://juejin.cn/post/7172464885492613128)

[Android源码分析 - Activity启动流程（下）](https://juejin.cn/post/7195458962328649788)

# 触发销毁

既然要分析`Activity`销毁流程，那我们就从最常见的入口`Activity.finish`入手

```java
public void finish() {
    finish(DONT_FINISH_TASK_WITH_ACTIVITY);
}
```

默认的`finish`方法调用了另一个同名重载方法，接受一个int类型的参数表明是否需要在销毁`Activity`的同时销毁`Task`，该参数有以下三种：

- `DONT_FINISH_TASK_WITH_ACTIVITY`：默认参数，表示在销毁`Activity`的时候不要销毁`Task`

- `FINISH_TASK_WITH_ROOT_ACTIVITY`：当`Activity`为跟`Activity`的时候，销毁的同时销毁`Task`，同时这个任务也会从最近任务中移除

- `FINISH_TASK_WITH_ACTIVITY`：销毁`Activity`的时候同时销毁`Task`，但不会从最近任务中移除

```java
private void finish(int finishTask) {
    if (mParent == null) {
        //当finish后才可能会触发onActivityResult回调
        //这里准备将result返回给之前调用startActivityForResult的Activity
        int resultCode;
        Intent resultData;
        synchronized (this) {
            resultCode = mResultCode;
            resultData = mResultData;
        }
        try {
            //两个Activity可能处于不同进程中，做进程间通信的准备
            if (resultData != null) {
                resultData.prepareToLeaveProcess(this);
            }
            //调用ATMS销毁Activity
            if (ActivityTaskManager.getService()
                    .finishActivity(mToken, resultCode, resultData, finishTask)) {
                mFinished = true;
            }
        } catch (RemoteException e) {
            // Empty
        }
    } else {
        mParent.finishFromChild(this);
    }

    // Activity was launched when user tapped a link in the Autofill Save UI - Save UI must
    // be restored now.
    if (mIntent != null && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN)) {
        restoreAutofillSaveUi();
    }
}
```

`onActivityResult`回调是在对应`Activity` `resume`时才可能触发，具体过程后面会分析，将`ActivityRecord.Token`和`Result`作为参数调用`ATMS.finishActivity`方法

```java
public final boolean finishActivity(IBinder token, int resultCode, Intent resultData,
        int finishTask) {
    // Refuse possible leaked file descriptors
    //回传的ResultIntent中不允许包含fd，防止泄漏
    if (resultData != null && resultData.hasFileDescriptors()) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    final ActivityRecord r;
    synchronized (mGlobalLock) {
        //获取ActivityRecord并保证其在栈中
        r = ActivityRecord.isInStackLocked(token);
        //为null说明已被移出ActivityStack，视作已被finish
        if (r == null) {
            return true;
        }
    }

    // Carefully collect grants without holding lock
    //检查调用方（即待finish的Activity）是否能授予result所对应Activity package访问uri的权限
    final NeededUriGrants resultGrants = collectGrants(resultData, r.resultTo);

    synchronized (mGlobalLock) {
        // Sanity check in case activity was removed before entering global lock.
        if (!r.isInHistory()) {
            return true;
        }

        // Keep track of the root activity of the task before we finish it
        final Task tr = r.getTask();
        final ActivityRecord rootR = tr.getRootActivity();
        // Do not allow task to finish if last task in lockTask mode. Launchable priv-apps can
        // finish.
        //LockTask模式下，如果此为最后一个Task，则不允许被销毁
        //详见：https://developer.android.com/work/dpc/dedicated-devices/lock-task-mode
        if (getLockTaskController().activityBlockedFromFinish(r)) {
            return false;
        }

        // TODO: There is a dup. of this block of code in ActivityStack.navigateUpToLocked
        // We should consolidate.
        //IActivityController分发Activity状态变化
        if (mController != null) {
            // Find the first activity that is not finishing.
            //寻找该Activity销毁后的下一个顶层Activity
            final ActivityRecord next =
                    r.getRootTask().topRunningActivity(token, INVALID_TASK_ID);
            if (next != null) {
                // ask watcher if this is allowed
                boolean resumeOK = true;
                try {
                    resumeOK = mController.activityResuming(next.packageName);
                } catch (RemoteException e) {
                    mController = null;
                    Watchdog.getInstance().setActivityController(null);
                }

                if (!resumeOK) {
                    return false;
                }
            }
        }

        // note down that the process has finished an activity and is in background activity
        // starts grace period
        //设置Activity销毁的最新时间
        if (r.app != null) {
            r.app.setLastActivityFinishTimeIfNeeded(SystemClock.uptimeMillis());
        }

        final long origId = Binder.clearCallingIdentity();
        try {
            boolean res;
            final boolean finishWithRootActivity =
                    finishTask == Activity.FINISH_TASK_WITH_ROOT_ACTIVITY;
            if (finishTask == Activity.FINISH_TASK_WITH_ACTIVITY
                    || (finishWithRootActivity && r == rootR)) { //需要同时销毁Task
                // If requested, remove the task that is associated to this activity only if it
                // was the root activity in the task. The result code and data is ignored
                // because we don't support returning them across task boundaries. Also, to
                // keep backwards compatibility we remove the task from recents when finishing
                // task with root activity.
                //移除Task
                mStackSupervisor.removeTask(tr, false /*killProcess*/,
                        finishWithRootActivity, "finish-activity");
                res = true;
                // Explicitly dismissing the activity so reset its relaunch flag.
                r.mRelaunchReason = RELAUNCH_REASON_NONE;
            } else { //不需要同时销毁Task
                r.finishIfPossible(resultCode, resultData, resultGrants,
                        "app-request", true /* oomAdj */);
                res = r.finishing;
            }
            return res;
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
}
```

这个方法里分了两个case，当需要同时销毁`Task`的时候，直接调用`ActivityStackSupervisor.removeTask`，当不需要同时销毁`Task`的时候，调用`ActivityRecord.finishIfPossible`

我们先看需要同时销毁`Task`的case

```java
void removeTask(Task task, boolean killProcess, boolean removeFromRecents, String reason) {
    if (task.mInRemoveTask) {
        // Prevent recursion.
        return;
    }
    task.mInRemoveTask = true;
    try {
        //执行Task移除操作
        task.performClearTask(reason);
        //对Task执行杀进程，从最近任务列表移除等操作
        cleanUpRemovedTaskLocked(task, killProcess, removeFromRecents);
        //关闭LockTask模式
        mService.getLockTaskController().clearLockedTask(task);
        //通知Task状态发生变化
        mService.getTaskChangeNotificationController().notifyTaskStackChanged();
        //将最近任务持久化保存
        if (task.isPersistable) {
            mService.notifyTaskPersisterLocked(null, true);
        }
    } finally {
        task.mInRemoveTask = false;
    }
}
```

本篇文章我们主要关注的是`Activity`销毁流程，至于进程的关闭，最近任务列表的更新我们在这里就不关心了，而这里`Activity`销毁的重点在于`Task.performClearTask`方法

```java
/** Completely remove all activities associated with an existing task. */
void performClearTask(String reason) {
    // Broken down into to cases to avoid object create due to capturing mStack.
    if (getStack() == null) {
        forAllActivities((r) -> {
            if (r.finishing) return;
            // Task was restored from persistent storage.
            r.takeFromHistory();
            removeChild(r);
        });
    } else {
        forAllActivities((r) -> {
            if (r.finishing) return;
            // TODO: figure-out how to avoid object creation due to capture of reason variable.
            r.finishIfPossible(Activity.RESULT_CANCELED,
                    null /* resultData */, null /* resultGrants */, reason, false /* oomAdj */);
        });
    }
}
```

我们看后半部分代码可以发现，这个方法对`Task`中所有未销毁的`Activity`都执行了`ActivityRecord.finishIfPossible`方法，这样路径就和上面`ATMS.finishActivity`方法中第二个case统一了

```java
/**
 * Finish activity if possible. If activity was resumed - we must first pause it to make the
 * activity below resumed. Otherwise we will try to complete the request immediately by calling
 * {@link #completeFinishing(String)}.
 * @return One of {@link FinishRequest} values:
 * {@link #FINISH_RESULT_REMOVED} if this activity has been removed from the history list.
 * {@link #FINISH_RESULT_REQUESTED} if removal process was started, but it is still in the list
 * and will be removed from history later.
 * {@link #FINISH_RESULT_CANCELLED} if activity is already finishing or in invalid state and the
 * request to finish it was not ignored.
 */
@FinishRequest int finishIfPossible(int resultCode, Intent resultData,
        NeededUriGrants resultGrants, String reason, boolean oomAdj) {

    //防止重复销毁
    if (finishing) {
        return FINISH_RESULT_CANCELLED;
    }

    //此Activity不在任务栈中
    if (!isInStackLocked()) {
        return FINISH_RESULT_CANCELLED;
    }

    final ActivityStack stack = getRootTask();
    //应该调整顶部Activity
    final boolean mayAdjustTop = (isState(RESUMED) || stack.mResumedActivity == null)
            && stack.isFocusedStackOnDisplay();
    //应该调整全局焦点
    final boolean shouldAdjustGlobalFocus = mayAdjustTop
            // It must be checked before {@link #makeFinishingLocked} is called, because a stack
            // is not visible if it only contains finishing activities.
            && mRootWindowContainer.isTopDisplayFocusedStack(stack);

    //暂停布局工作
    mAtmService.deferWindowLayout();
    try {
        //设置当前Activity状态为finishing
        makeFinishingLocked();
        // Make a local reference to its task since this.task could be set to null once this
        // activity is destroyed and detached from task.
        final Task task = getTask();
        //获取上一个ActivityRecord
        ActivityRecord next = task.getActivityAbove(this);
        //传递FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET：重置该Task时清除此Activity
        if (next != null) {
            if ((intent.getFlags() & Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET) != 0) {
                // If the caller asked that this activity (and all above it)
                // be cleared when the task is reset, don't lose that information,
                // but propagate it up to the next activity.
                next.intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET);
            }
        }

        //暂停输入事件分发
        pauseKeyDispatchingLocked();

        // We are finishing the top focused activity and its task has nothing to be focused so
        // the next focusable task should be focused.
        //应该调整顶部Activity，但此Task没有Activity可以被运行在顶部，将焦点转移至下一个可聚焦的Task
        if (mayAdjustTop && ((ActivityStack) task).topRunningActivity(true /* focusableOnly */)
                == null) {
            task.adjustFocusToNextFocusableTask("finish-top", false /* allowFocusSelf */,
                        shouldAdjustGlobalFocus);
        }

        //将Result信息写入到对应ActivityRecord中，待后面resume的时候触发onActivityResult回调
        finishActivityResults(resultCode, resultData, resultGrants);

        //终止Task
        final boolean endTask = task.getActivityBelow(this) == null
                && !task.isClearingToReuseTask();
        final int transit = endTask ? TRANSIT_TASK_CLOSE : TRANSIT_ACTIVITY_CLOSE;
        if (isState(RESUMED)) {
            if (endTask) {
                //通知Task移除已开始
                mAtmService.getTaskChangeNotificationController().notifyTaskRemovalStarted(
                        task.getTaskInfo());
            }
            // Prepare app close transition, but don't execute just yet. It is possible that
            // an activity that will be made resumed in place of this one will immediately
            // launch another new activity. In this case current closing transition will be
            // combined with open transition for the new activity.
            //准备Activity转场动画
            mDisplayContent.prepareAppTransition(transit, false);

            // When finishing the activity preemptively take the snapshot before the app window
            // is marked as hidden and any configuration changes take place
            //更新Task快照
            if (mAtmService.mWindowManager.mTaskSnapshotController != null) {
                final ArraySet<Task> tasks = Sets.newArraySet(task);
                mAtmService.mWindowManager.mTaskSnapshotController.snapshotTasks(tasks);
                mAtmService.mWindowManager.mTaskSnapshotController
                        .addSkipClosingAppSnapshotTasks(tasks);
            }

            // Tell window manager to prepare for this one to be removed.
            //设置可见性
            setVisibility(false);

            if (stack.mPausingActivity == null) {
                //开始暂停此Activity
                stack.startPausingLocked(false /* userLeaving */, false /* uiSleeping */,
                        null /* resuming */);
            }

            if (endTask) {
                //屏幕固定功能
                mAtmService.getLockTaskController().clearLockedTask(task);
                // This activity was in the top focused stack and this is the last activity in
                // that task, give this activity a higher layer so it can stay on top before the
                // closing task transition be executed.
                //更新窗口层级
                if (mayAdjustTop) {
                    mNeedsZBoost = true;
                    mDisplayContent.assignWindowLayers(false /* setLayoutNeeded */);
                }
            }
        } else if (!isState(PAUSING)) {
            ... //正常不会进入此case
        }

        return FINISH_RESULT_REQUESTED;
    } finally {
        //恢复布局工作
        mAtmService.continueWindowLayout();
    }
}
```

这个方法中，我们需要关注一下对于`Result`信息的处理，这里调用了`finishActivityResults`方法，将`Result`信息写入到对应`ActivityRecord`中，待后面`resume`的时候触发`onActivityResult`回调

```java
/**
 * Sets the result for activity that started this one, clears the references to activities
 * started for result from this one, and clears new intents.
 */
private void finishActivityResults(int resultCode, Intent resultData,
        NeededUriGrants resultGrants) {
    // Send the result if needed
    if (resultTo != null) {
        if (resultTo.mUserId != mUserId) {
            if (resultData != null) {
                resultData.prepareToLeaveUser(mUserId);
            }
        }
        if (info.applicationInfo.uid > 0) {
            mAtmService.mUgmInternal.grantUriPermissionUncheckedFromIntent(resultGrants,
                    resultTo.getUriPermissionsLocked());
        }
        resultTo.addResultLocked(this, resultWho, requestCode, resultCode, resultData);
        resultTo = null;
    }

    // Make sure this HistoryRecord is not holding on to other resources,
    // because clients have remote IPC references to this object so we
    // can't assume that will go away and want to avoid circular IPC refs.
    results = null;
    pendingResults = null;
    newIntents = null;
    setSavedState(null /* savedState */);
}

//将Result结果添加到results列表中
void addResultLocked(ActivityRecord from, String resultWho,
        int requestCode, int resultCode,
        Intent resultData) {
    ActivityResult r = new ActivityResult(from, resultWho,
            requestCode, resultCode, resultData);
    if (results == null) {
        results = new ArrayList<ResultInfo>();
    }
    results.add(r);
}
```

这个方法很简单，就是将`Result`信息添加到`ActivityRecord.results`列表中

然后我们继续沿着`finish`主线链路走，后面有一个`isState`的判断，正常来说，`ActivityRecord`的`state`应该为`RESUMED`，具体为什么我们可以回顾一下之前分析的`Activity`启动流程，在`ActivityStackSupervisor.realStartActivityLocked`方法最后，会调用`ActivityStack.minimalResumeActivityLocked`，在这个方法中，会将`ActivityRecord`的`state`设置为`RESUMED`，由于`ClientTransaction`的执行是通过`Handler.sendMessage`进行的，所以早在`Activity` `onCreate`之前，`ActivityRecord`的状态就已经被设为了`RESUMED`

根据以上分析，我们会走进`isState(RESUMED)`这个case中，接着调用`ActivityStack.startPausingLocked`方法暂停`Activity`

```java
/**
 * Start pausing the currently resumed activity.  It is an error to call this if there
 * is already an activity being paused or there is no resumed activity.
 *
 * @param userLeaving True if this should result in an onUserLeaving to the current activity.
 * @param uiSleeping True if this is happening with the user interface going to sleep (the
 * screen turning off).
 * @param resuming The activity we are currently trying to resume or null if this is not being
 *                 called as part of resuming the top activity, so we shouldn't try to instigate
 *                 a resume here if not null.
 * @return Returns true if an activity now is in the PAUSING state, and we are waiting for
 * it to tell us when it is done.
 */
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
        ActivityRecord resuming) {
    //已有Activity正在暂停中
    if (mPausingActivity != null) {
        if (!shouldSleepActivities()) {
            // Avoid recursion among check for sleep and complete pause during sleeping.
            // Because activity will be paused immediately after resume, just let pause
            // be completed by the order of activity paused from clients.
            completePauseLocked(false, resuming);
        }
    }
    //上一个已resume的Activity
    ActivityRecord prev = mResumedActivity;

    //既没有已resume的Activity，也没有正在resume的Activity
    //从栈顶找一个Activity恢复
    if (prev == null) {
        if (resuming == null) {
            mRootWindowContainer.resumeFocusedStacksTopActivities();
        }
        return false;
    }

    //不能暂停一个正在resume的Activity
    if (prev == resuming) {
        return false;
    }

    //设置各种状态
    mPausingActivity = prev;
    mLastPausedActivity = prev;
    mLastNoHistoryActivity = prev.isNoHistory() ? prev : null;
    prev.setState(PAUSING, "startPausingLocked");
    prev.getTask().touchActiveTime();
    clearLaunchTime(prev);

    //更新统计信息
    mAtmService.updateCpuStats();

    boolean pauseImmediately = false;
    ... //当前流程下pauseImmediately始终为false

    if (prev.attachedToProcess()) {
        try {
            //调度Pause生命周期事务
            mAtmService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
                    prev.appToken, PauseActivityItem.obtain(prev.finishing, userLeaving,
                            prev.configChangeFlags, pauseImmediately));
        } catch (Exception e) {
            // Ignore exception, if process died other code will cleanup.
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
    } else {
        mPausingActivity = null;
        mLastPausedActivity = null;
        mLastNoHistoryActivity = null;
    }

    // If we are not going to sleep, we want to ensure the device is
    // awake until the next activity is started.
    //获取Wakelock，确保设备awake状态直到下一个Activity启动
    if (!uiSleeping && !mAtmService.isSleepingOrShuttingDownLocked()) {
        mStackSupervisor.acquireLaunchWakelock();
    }

    if (mPausingActivity != null) {
        // Have the window manager pause its key dispatching until the new
        // activity has started.  If we're pausing the activity just because
        // the screen is being turned off and the UI is sleeping, don't interrupt
        // key dispatch; the same activity will pick it up again on wakeup.
        if (!uiSleeping) {
            //暂停输入事件分发
            prev.pauseKeyDispatchingLocked();
        }

        if (pauseImmediately) { //不会进入此case
            // If the caller said they don't want to wait for the pause, then complete
            // the pause now.
            completePauseLocked(false, resuming);
            return false;
        } else {
            //设置超时监听（500ms内没有完成便视为超时）
            prev.schedulePauseTimeout();
            return true;
        }

    } else {
        // This activity failed to schedule the
        // pause, so just treat it as being paused now.
        //未能成功暂停此Activity，从栈顶找一个Activity恢复
        if (resuming == null) {
            mRootWindowContainer.resumeFocusedStacksTopActivities();
        }
        return false;
    }
}
```

可以看到，和`Activity`启动流程类似，该方法里面调用了`ClientLifecycleManager.scheduleTransaction`方法来调度`Activity`暂停的生命周期，具体是怎样调度的可以看我之前的文章 [Android源码分析 - Activity启动流程（下）](https://juejin.cn/post/7195458962328649788#heading-5)，里面分析了`ClientTransaction`事务是怎么被调度执行的

了解完后我们就可以知道，生命周期事务的执行也就相当于分别调用`ActivityLifecycleItem`的`preExecute`、`execute`、`postExecute`方法，而`PauseActivityItem`没有重写`preExecute`方法，所以我们就依次分析其`execute`、`postExecute`方法就好了

```java
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
            "PAUSE_ACTIVITY_ITEM");
}
```

`ClientTransactionHandler`这个我们之前说过，这是一个抽象类，被`ActivityThread`继承实现，所以这里实际上就是调用`ActivityThread.handlePauseActivity`方法

# handlePauseActivity

```java
public void handlePauseActivity(IBinder token, boolean finished, boolean userLeaving,
        int configChanges, PendingTransactionActions pendingActions, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        ...
        r.activity.mConfigChangeFlags |= configChanges;
        performPauseActivity(r, finished, reason, pendingActions);

        // Make sure any pending writes are now committed.
        //确保所有全局任务都被处理完成
        if (r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }
        //更新标记
        mSomeActivitiesChanged = true;
    }
}

/**
 * Pause the activity.
 * @return Saved instance state for pre-Honeycomb apps if it was saved, {@code null} otherwise.
 */
private Bundle performPauseActivity(ActivityClientRecord r, boolean finished, String reason,
        PendingTransactionActions pendingActions) {
    ... //异常状态检查
    if (finished) {
        r.activity.mFinished = true;
    }

    // Pre-Honeycomb apps always save their state before pausing
    //是否需要保存状态信息（Android 3.0前无论是否finish都会触发保存）
    final boolean shouldSaveState = !r.activity.mFinished && r.isPreHoneycomb();
    if (shouldSaveState) {
        //回调Activity的onSaveInstanceState方法
        callActivityOnSaveInstanceState(r);
    }

    performPauseActivityIfNeeded(r, reason);

    ...//回调OnActivityPausedListener，目前看来只有NFC部分有注册这个回调

    ... //Android 3.0之前的特殊处理

    //返回保存状态的Bundle
    return shouldSaveState ? r.state : null;
}

private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
    //已暂停，直接返回
    if (r.paused) {
        // You are already paused silly...
        return;
    }

    // Always reporting top resumed position loss when pausing an activity. If necessary, it
    // will be restored in performResumeActivity().
    //报告resume状态变更
    reportTopResumedActivityChanged(r, false /* onTop */, "pausing");

    try {
        r.activity.mCalled = false;
        //回调Activity的onPause方法
        mInstrumentation.callActivityOnPause(r.activity);
        if (!r.activity.mCalled) {
            //必须要调用super.onPause方法
            throw new SuperNotCalledException("Activity " + safeToComponentShortString(r.intent)
                    + " did not call through to super.onPause()");
        }
    } catch ...
    //设置状态
    r.setState(ON_PAUSE);
}
```

这一条调用链路看下来还是很简单的，和之前我们分析过的其他生命周期调用是一个套路，这里显示调用了`callActivityOnSaveInstanceState`方法保存状态信息

```java
private void callActivityOnSaveInstanceState(ActivityClientRecord r) {
    r.state = new Bundle();
    r.state.setAllowFds(false);
    if (r.isPersistable()) {
        r.persistentState = new PersistableBundle();
        mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state,
                r.persistentState);
    } else {
        mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state);
    }
}
```

通过`Instrumentation`调用`Activity.performSaveInstanceState`方法

```java
final void performSaveInstanceState(@NonNull Bundle outState) {
    //分发PreSaveInstanceState事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPreSaveInstanceState回调
    dispatchActivityPreSaveInstanceState(outState);
    //回调onSaveInstanceState
    onSaveInstanceState(outState);
    //保存受管理的Dialog的状态
    saveManagedDialogs(outState);
    //共享元素动画相关
    mActivityTransitionState.saveState(outState);
    //保存权限请求状态
    storeHasCurrentPermissionRequest(outState);
    //分发PostSaveInstanceState事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPostSaveInstanceState回调
    dispatchActivityPostSaveInstanceState(outState);
}
```

最终回调`Activity.onSaveInstanceState`方法

```java
protected void onSaveInstanceState(@NonNull Bundle outState) {
    //保存窗口信息
    outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());

    outState.putInt(LAST_AUTOFILL_ID, mLastAutofillId);
    //保存Fragment状态
    Parcelable p = mFragments.saveAllState();
    if (p != null) {
        outState.putParcelable(FRAGMENTS_TAG, p);
    }
    //自动填充相关
    if (mAutoFillResetNeeded) {
        outState.putBoolean(AUTOFILL_RESET_NEEDED, true);
        getAutofillManager().onSaveInstanceState(outState);
    }
    //分发SaveInstanceState事件，执行所有注册的ActivityLifecycleCallbacks的onActivitySaveInstanceState回调
    dispatchActivitySaveInstanceState(outState);
}
```

保存状态的流程就基本完成了，我们再回过头来看`onPause`的触发

在上面`performPauseActivityIfNeeded`方法中有一行代码调用了`Instrumentation.callActivityOnPause`方法，
通过`Instrumentation`调用了`Activity.performPause`方法

```java
final void performPause() {
    //分发PrePaused事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPrePaused回调
    dispatchActivityPrePaused();
    mDoReportFullyDrawn = false;
    //FragmentManager分发pause状态
    mFragments.dispatchPause();
    mCalled = false;
    //回调onPause
    onPause();
    mResumed = false;
    //Target Sdk 9以上（Android 2.3）需要保证在onPause中调用super.onPause方法
    if (!mCalled && getApplicationInfo().targetSdkVersion
            >= android.os.Build.VERSION_CODES.GINGERBREAD) {
        throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onPause()");
    }
    //分发PostPaused事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPostPaused回调
    dispatchActivityPostPaused();
}
```

执行`onPause`回调

```java
protected void onPause() {
    //分发Paused事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPaused回调
    dispatchActivityPaused();
    //自动填充相关
    if (mAutoFillResetNeeded) {
        if (!mAutoFillIgnoreFirstResumePause) {
            View focus = getCurrentFocus();
            if (focus != null && focus.canNotifyAutofillEnterExitEvent()) {
                getAutofillManager().notifyViewExited(focus);
            }
        } else {
            // reset after first pause()
            mAutoFillIgnoreFirstResumePause = false;
        }
    }
    //内容捕获服务
    notifyContentCaptureManagerIfNeeded(CONTENT_CAPTURE_PAUSE);
    //super.onPause标注为已被执行
    mCalled = true;
}
```

到此为止，`Activity`的`onPause`生命周期已经基本走完了，此时我们再回到`PauseActivityItem.postExecute`方法中做一些善后处理

```java
public void postExecute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    //mDontReport为我们之前obtain方法中传入的pauseImmediately参数，始终为false
    if (mDontReport) {
        return;
    }
    try {
        // TODO(lifecycler): Use interface callback instead of AMS.
        //调用ATMS.activityPaused方法
        ActivityTaskManager.getService().activityPaused(token);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
}
```

这里调用`ATMS.activityPaused`方法回到`system_server`进程处理`Activity`暂停后的事项

```java
public final void activityPaused(IBinder token) {
    final long origId = Binder.clearCallingIdentity();
    synchronized (mGlobalLock) {
        //通过ActivityRecord.Token获取ActivityRecord
        final ActivityRecord r = ActivityRecord.forTokenLocked(token);
        if (r != null) {
            r.activityPaused(false);
        }
    }
    Binder.restoreCallingIdentity(origId);
}
```

调用`ActivityRecord.activityPaused`方法继续处理

```java
void activityPaused(boolean timeout) {
    final ActivityStack stack = getStack();

    if (stack != null) {
        //移除超时监听
        removePauseTimeout();

        if (stack.mPausingActivity == this) {
            //暂停布局工作
            mAtmService.deferWindowLayout();
            try {
                stack.completePauseLocked(true /* resumeNext */, null /* resumingActivity */);
            } finally {
                //恢复布局工作
                mAtmService.continueWindowLayout();
            }
            return;
        } else { //暂停Activity失败
            if (isState(PAUSING)) {
                setState(PAUSED, "activityPausedLocked");
                if (finishing) {
                    completeFinishing("activityPausedLocked");
                }
            }
        }
    }

    //更新Activity可见性
    mRootWindowContainer.ensureActivitiesVisible(null, 0, !PRESERVE_WINDOWS);
}
```

正常情况下会进入到`ActivityStack.completePauseLocked`方法中，但在暂停`Activity`失败的情况下，如果当前状态为`PAUSING`，则直接将其状态置为`PAUSED`已暂停，如果被标记为`finishing`，则会调用`ActivityRecord.completeFinishing`继续`finish`流程，这其实和正常情况下的调用链路差不多，具体我们往下就能看到

```java
void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
    ActivityRecord prev = mPausingActivity;

    if (prev != null) {
        prev.setWillCloseOrEnterPip(false);
        //之前的状态是否为正在停止
        final boolean wasStopping = prev.isState(STOPPING);
        //设置状态为已暂停
        prev.setState(PAUSED, "completePausedLocked");
        if (prev.finishing) {
            //继续finish流程
            prev = prev.completeFinishing("completePausedLocked");
        } else if (prev.hasProcess()) {
            //Configuration发生变化时可能会设置这个flag
            if (prev.deferRelaunchUntilPaused) {
                // Complete the deferred relaunch that was waiting for pause to complete.
                //等待暂停完成后relaunch Activity
                prev.relaunchActivityLocked(prev.preserveWindowOnDeferredRelaunch);
            } else if (wasStopping) {
                // We are also stopping, the stop request must have gone soon after the pause.
                // We can't clobber it, because the stop confirmation will not be handled.
                // We don't need to schedule another stop, we only need to let it happen.
                //之前的状态为正在停止，将状态置回即可
                prev.setState(STOPPING, "completePausedLocked");
            } else if (!prev.mVisibleRequested || shouldSleepOrShutDownActivities()) {
                // Clear out any deferred client hide we might currently have.
                prev.setDeferHidingClient(false);
                // If we were visible then resumeTopActivities will release resources before
                // stopping.
                //添加到stop列表中等待空闲时执行stop
                prev.addToStopping(true /* scheduleIdle */, false /* idleDelayed */,
                        "completePauseLocked");
            }
        } else {
            //App在pause过程中死亡
            prev = null;
        }
        // It is possible the activity was freezing the screen before it was paused.
        // In that case go ahead and remove the freeze this activity has on the screen
        // since it is no longer visible.
        if (prev != null) {
            //停止屏幕冻结
            prev.stopFreezingScreenLocked(true /*force*/);
        }
        //Activity暂停完毕
        mPausingActivity = null;
    }

    //恢复前一个顶层Activity
    if (resumeNext) {
        final ActivityStack topStack = mRootWindowContainer.getTopDisplayFocusedStack();
        if (topStack != null && !topStack.shouldSleepOrShutDownActivities()) {
            mRootWindowContainer.resumeFocusedStacksTopActivities(topStack, prev, null);
        } else {
            checkReadyForSleep();
            final ActivityRecord top = topStack != null ? topStack.topRunningActivity() : null;
            if (top == null || (prev != null && top != prev)) {
                // If there are no more activities available to run, do resume anyway to start
                // something. Also if the top activity on the stack is not the just paused
                // activity, we need to go ahead and resume it to ensure we complete an
                // in-flight app switch.
                mRootWindowContainer.resumeFocusedStacksTopActivities();
            }
        }
    }

    if (prev != null) {
        //恢复按键分发
        prev.resumeKeyDispatchingLocked();
        ... //更新统计信息
    }

    //更新Activity可见性
    mRootWindowContainer.ensureActivitiesVisible(resuming, 0, !PRESERVE_WINDOWS);

    // Notify when the task stack has changed, but only if visibilities changed (not just
    // focus). Also if there is an active pinned stack - we always want to notify it about
    // task stack changes, because its positioning may depend on it.
    //通知Task状态发生变化
    if (mStackSupervisor.mAppVisibilitiesChangedSinceLastPause
            || (getDisplayArea() != null && getDisplayArea().hasPinnedTask())) {
        mAtmService.getTaskChangeNotificationController().notifyTaskStackChanged();
        mStackSupervisor.mAppVisibilitiesChangedSinceLastPause = false;
    }
}
```

可以看到，无论暂停成功与否，最后都会走到`ActivityRecord.completeFinishing`方法中

```java
/**
 * Complete activity finish request that was initiated earlier. If the activity is still
 * pausing we will wait for it to complete its transition. If the activity that should appear in
 * place of this one is not visible yet - we'll wait for it first. Otherwise - activity can be
 * destroyed right away.
 * @param reason Reason for finishing the activity.
 * @return Flag indicating whether the activity was removed from history.
 */
ActivityRecord completeFinishing(String reason) {
    ... //状态检查

    final boolean isCurrentVisible = mVisibleRequested || isState(PAUSED);
    if (isCurrentVisible) {
        ... //更新Activity可见性
    }

    boolean activityRemoved = false;

    // If this activity is currently visible, and the resumed activity is not yet visible, then
    // hold off on finishing until the resumed one becomes visible.
    // The activity that we are finishing may be over the lock screen. In this case, we do not
    // want to consider activities that cannot be shown on the lock screen as running and should
    // proceed with finishing the activity if there is no valid next top running activity.
    // Note that if this finishing activity is floating task, we don't need to wait the
    // next activity resume and can destroy it directly.
    // TODO(b/137329632): find the next activity directly underneath this one, not just anywhere
    final ActivityRecord next = getDisplayArea().topRunningActivity(
            true /* considerKeyguardState */);
    // isNextNotYetVisible is to check if the next activity is invisible, or it has been
    // requested to be invisible but its windows haven't reported as invisible.  If so, it
    // implied that the current finishing activity should be added into stopping list rather
    // than destroy immediately.
    final boolean isNextNotYetVisible = next != null
            && (!next.nowVisible || !next.mVisibleRequested);

    //如果此Activity当前可见，而要恢复的Activity还不可见，则推迟finish，直到要恢复的Activity可见为止
    if (isCurrentVisible && isNextNotYetVisible) {
        // Add this activity to the list of stopping activities. It will be processed and
        // destroyed when the next activity reports idle.
        //添加到stop列表中等待空闲时执行stop
        addToStopping(false /* scheduleIdle */, false /* idleDelayed */,
                "completeFinishing");
        //设置状态为stop中
        setState(STOPPING, "completeFinishing");
    } else if (addToFinishingAndWaitForIdle()) {
        // We added this activity to the finishing list and something else is becoming resumed.
        // The activity will complete finishing when the next activity reports idle. No need to
        // do anything else here.
        //将此Activity添加到待finish列表中，等待空闲时执行finish
    } else {
        // Not waiting for the next one to become visible, and nothing else will be resumed in
        // place of this activity - requesting destruction right away.
        //立刻销毁此Activity
        activityRemoved = destroyIfPossible(reason);
    }

    return activityRemoved ? null : this;
}
```

对于非锁屏状态且当前要销毁的`Activity`在前台的情况下，该`Activity`可见而待恢复的`Activity`尚不可见，此时优先完成待恢复`Activity`的`resume`生命周期，等到之后空闲再去处理待销毁`Activity`的`destroy`生命周期

所以在面试中常问的`Activity`从`B`返回到`A`的生命周期顺序我们从这里就可以看出来，理解后我们就不用去死记硬背了：

`B.onPause` -> `A.onRestart` -> `A.onResume` -> `B.onStop` -> `B.onDestory`

对于锁屏状态或者要销毁的`Activity`不在前台的情况下，由于不需要立刻恢复`Activity`，所以可能会直接处理待销毁`Activity`的`destroy`生命周期

我们以第一种当前要销毁的`Activity`在前台的情况分析，此时会将这个`Activity`添加到`stop`列表中，并将状态设置为`STOPPING`，之后返回到`ActivityStack.completePauseLocked`方法中，继续执行`resumeNext`工作

在`resumeNext`中会调用`RootWindowContainer.resumeFocusedStacksTopActivities`方法恢复栈顶`Activity`，由于这个方法之前已经在 
[Android源码分析 - Activity启动流程（上）](https://juejin.cn/post/7130182223231188999#heading-6) 中分析过了，这里就不再赘述了，我们还是将目光放在销毁流程上

通过之前的文章，我们知道恢复`Activity`会调用到`ActivityThread.handleResumeActivity`方法，而当`Activity`恢复完毕后，此方法最后一行会向`MessageQueue`添加一个`IdleHandler`，关于`IdleHandler`这里就不再介绍了，这是每位`Android`开发都应该了解的东西

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ...
    r.nextIdle = mNewActivities;
    mNewActivities = r;
    Looper.myQueue().addIdleHandler(new Idler());
}
```

这里的`Idler`是`ActivityThread`的一个内部类

```java
private class Idler implements MessageQueue.IdleHandler {
    @Override
    public final boolean queueIdle() {
        ActivityClientRecord a = mNewActivities;
        ...
        if (a != null) {
            mNewActivities = null;
            IActivityTaskManager am = ActivityTaskManager.getService();
            ActivityClientRecord prev;
            //遍历整条ActivityClientRecord.nextIdle链，依次调用ATMS.activityIdle
            do {
                if (a.activity != null && !a.activity.mFinished) {
                    try {
                        am.activityIdle(a.token, a.createdConfig, stopProfiling);
                        a.createdConfig = null;
                    } catch (RemoteException ex) {
                        throw ex.rethrowFromSystemServer();
                    }
                }
                prev = a;
                a = a.nextIdle;
                prev.nextIdle = null;
            } while (a != null);
        }
        ...
        return false;
    }
}
```

这里会遍历整个进程内所有的`ActivityClientRecord`，并依次调用`ATMS.activityIdle`方法

```java
public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
    ...
    final ActivityRecord r = ActivityRecord.forTokenLocked(token);
    if (r == null) {
        return;
    }
    mStackSupervisor.activityIdleInternal(r, false /* fromTimeout */,
            false /* processPausingActivities */, config);
    ...
}
```

从`ActivityRecord.Token`获取到`ActivityRecord`，接着调用`ActivityStackSupervisor.activityIdleInternal`方法

```java
void activityIdleInternal(ActivityRecord r, boolean fromTimeout,
        boolean processPausingActivities, Configuration config) {
    ...
    // Atomically retrieve all of the other things to do.
    processStoppingAndFinishingActivities(r, processPausingActivities, "idle");
    ...
}
```

这里我们只需要重点关注`processStoppingAndFinishingActivities`这一个方法，从方法名我们也能看出来，它是用来处理`Activity` `stop`或`destroy`的

```java
/**
 * Processes the activities to be stopped or destroyed. This should be called when the resumed
 * activities are idle or drawn.
 */
private void processStoppingAndFinishingActivities(ActivityRecord launchedActivity,
        boolean processPausingActivities, String reason) {
    // Stop any activities that are scheduled to do so but have been waiting for the transition
    // animation to finish.
    ArrayList<ActivityRecord> readyToStopActivities = null;
    for (int i = mStoppingActivities.size() - 1; i >= 0; --i) {
        final ActivityRecord s = mStoppingActivities.get(i);
        final boolean animating = s.isAnimating(TRANSITION | PARENTS,
                ANIMATION_TYPE_APP_TRANSITION | ANIMATION_TYPE_RECENTS);
        //不在动画中或者ATMS服务正在关闭
        if (!animating || mService.mShuttingDown) {
            //跳过正在pause的Activitiy
            if (!processPausingActivities && s.isState(PAUSING)) {
                // Defer processing pausing activities in this iteration and reschedule
                // a delayed idle to reprocess it again
                removeIdleTimeoutForActivity(launchedActivity);
                scheduleIdleTimeout(launchedActivity);
                continue;
            }

            if (readyToStopActivities == null) {
                readyToStopActivities = new ArrayList<>();
            }
            //将准备好stop的Activitiy加入列表中
            readyToStopActivities.add(s);

            mStoppingActivities.remove(i);
        }
    }

    final int numReadyStops = readyToStopActivities == null ? 0 : readyToStopActivities.size();
    for (int i = 0; i < numReadyStops; i++) {
        final ActivityRecord r = readyToStopActivities.get(i);
        //Activity是否在任务栈中
        if (r.isInHistory()) {
            if (r.finishing) {
                // TODO(b/137329632): Wait for idle of the right activity, not just any.
                //被标记为finishing，尝试销毁Activity
                r.destroyIfPossible(reason);
            } else {
                //否则仅仅只是stop Activity
                r.stopIfPossible();
            }
        }
    }

    final int numFinishingActivities = mFinishingActivities.size();
    if (numFinishingActivities == 0) {
        return;
    }

    // Finish any activities that are scheduled to do so but have been waiting for the next one
    // to start.
    final ArrayList<ActivityRecord> finishingActivities = new ArrayList<>(mFinishingActivities);
    mFinishingActivities.clear();
    for (int i = 0; i < numFinishingActivities; i++) {
        final ActivityRecord r = finishingActivities.get(i);
        if (r.isInHistory()) {
            //立刻执行Activity的销毁流程
            r.destroyImmediately(true /* removeFromApp */, "finish-" + reason);
        }
    }
}
```

对于被标记为`finishing`的`Activity`，调用`destroyIfPossible`方法销毁

```java
/**
 * Destroy and cleanup the activity both on client and server if possible. If activity is the
 * last one left on display with home stack and there is no other running activity - delay
 * destroying it until the next one starts.
 */
boolean destroyIfPossible(String reason) {
    //设置状态
    setState(FINISHING, "destroyIfPossible");

    // Make sure the record is cleaned out of other places.
    //确保此Activity已从待stop列表中移除
    mStackSupervisor.mStoppingActivities.remove(this);

    final ActivityStack stack = getRootTask();
    final TaskDisplayArea taskDisplayArea = getDisplayArea();
    final ActivityRecord next = taskDisplayArea.topRunningActivity();
    final boolean isLastStackOverEmptyHome =
            next == null && stack.isFocusedStackOnDisplay()
                    && taskDisplayArea.getOrCreateRootHomeTask() != null;
    if (isLastStackOverEmptyHome) {
        // Don't destroy activity immediately if this is the last activity on the display and
        // the display contains home stack. Although there is no next activity at the moment,
        // another home activity should be started later. Keep this activity alive until next
        // home activity is resumed. This way the user won't see a temporary black screen.
        //如果Home栈存在且这是当前焦点栈中最后一个Activity，则不要立即销毁它
        //将此Activity添加到待finish列表中，等待空闲时执行finish
        addToFinishingAndWaitForIdle();
        return false;
    }
    //设置finishing标记（之前设过了，这里是重复设置）
    makeFinishingLocked();

    //立刻执行Activity的销毁流程
    final boolean activityRemoved = destroyImmediately(true /* removeFromApp */,
            "finish-imm:" + reason);

    // If the display does not have running activity, the configuration may need to be
    // updated for restoring original orientation of the display.
    //更新可见性和屏幕显示方向
    if (next == null) {
        mRootWindowContainer.ensureVisibilityAndConfig(next, getDisplayId(),
                false /* markFrozenIfConfigChanged */, true /* deferResume */);
    }
    //更新恢复栈顶Activity
    if (activityRemoved) {
        mRootWindowContainer.resumeFocusedStacksTopActivities();
    }

    return activityRemoved;
}
```

这里做了最后的一些判断，然后调用`destroyImmediately`方法，立刻执行Activity的销毁流程（这里和上一个方法`processStoppingAndFinishingActivities`中，待`finish`列表的处理是一样的）

```java
/**
 * Destroy the current CLIENT SIDE instance of an activity. This may be called both when
 * actually finishing an activity, or when performing a configuration switch where we destroy
 * the current client-side object but then create a new client-side object for this same
 * HistoryRecord.
 * Normally the server-side record will be removed when the client reports back after
 * destruction. If, however, at this point there is no client process attached, the record will
 * be removed immediately.
 *
 * @return {@code true} if activity was immediately removed from history, {@code false}
 * otherwise.
 */
boolean destroyImmediately(boolean removeFromApp, String reason) {
    //已经被销毁或正在被销毁，直接返回
    if (isState(DESTROYING, DESTROYED)) {
        return false;
    }

    boolean removedFromHistory = false;

    //清理工作
    cleanUp(false /* cleanServices */, false /* setState */);

    if (hasProcess()) {
        //清理更新工作
        if (removeFromApp) {
            app.removeActivity(this);
            if (!app.hasActivities()) {
                mAtmService.clearHeavyWeightProcessIfEquals(app);
                // Update any services we are bound to that might care about whether
                // their client may have activities.
                // No longer have activities, so update LRU list and oom adj.
                //更新进程信息
                app.updateProcessInfo(true /* updateServiceConnectionActivities */,
                        false /* activityChange */, true /* updateOomAdj */,
                        false /* addPendingTopUid */);
            }
        }

        boolean skipDestroy = false;

        try {
            //调度销毁生命周期事务
            mAtmService.getLifecycleManager().scheduleTransaction(app.getThread(), appToken,
                    DestroyActivityItem.obtain(finishing, configChangeFlags));
        } catch (Exception e) {
            // We can just ignore exceptions here...  if the process has crashed, our death
            // notification will clean things up.
            if (finishing) {
                //从历史任务中移除
                removeFromHistory(reason + " exceptionInScheduleDestroy");
                removedFromHistory = true;
                skipDestroy = true;
            }
        }

        nowVisible = false;

        // If the activity is finishing, we need to wait on removing it from the list to give it
        // a chance to do its cleanup.  During that time it may make calls back with its token
        // so we need to be able to find it on the list and so we don't want to remove it from
        // the list yet.  Otherwise, we can just immediately put it in the destroyed state since
        // we are not removing it from the list.
        if (finishing && !skipDestroy) {
            //设置状态
            setState(DESTROYING,
                    "destroyActivityLocked. finishing and not skipping destroy");
            //设置销毁超时回调
            mAtmService.mH.postDelayed(mDestroyTimeoutRunnable, DESTROY_TIMEOUT);
        } else {
            //设置状态
            setState(DESTROYED,
                    "destroyActivityLocked. not finishing or skipping destroy");
            app = null;
        }
    } else {
        // Remove this record from the history.
        if (finishing) {
            //没有绑定进程，从历史任务中移除
            removeFromHistory(reason + " hadNoApp");
            removedFromHistory = true;
        } else {
            //没有绑定进程且不在finishing中，直接设置状态为已被销毁
            setState(DESTROYED, "destroyActivityLocked. not finishing and had no app");
        }
    }

    configChangeFlags = 0;

    return removedFromHistory;
}
```

# scheduleTransaction

这个方法做了一些清理工作，重头戏在于调用了`ClientLifecycleManager.scheduleTransaction`方法调度销毁生命周期事务，接下来我们就重点分析这个事务的执行路径

`scheduleTransaction`的调用链路我们在 [Android源码分析 - Activity启动流程（下）](https://juejin.cn/post/7195458962328649788#heading-5) 中已经分析过了，这里我就简单的标注一下流程：

`ClientLifecycleManager.scheduleTransaction` ->
`ClientTransaction.schedule` ->
`ActivityThread.scheduleTransaction` ->
`ClientTransaction.preExecute` ->
`ActivityLifecycleItem.preExecute`->
`ActivityThread.sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction)` ->
`TransactionExecutor.execute` -> 
`TransactionExecutor.executeCallbacks` -> 
`TransactionExecutor.executeLifecycleState` ->
`TransactionExecutor.cycleToPath` ->
`ActivityLifecycleItem.execute` ->
`ActivityLifecycleItem.postExecute`

这里的链路基本上和`Activity`启动事务链路相差无几，甚至更短了（`Activity`销毁事务没有添加`callback`），所以没看过我上篇文章的强烈推荐去看一下，这里我就不再做分析了

我们从`TransactionExecutor.cycleToPath`开始，之前我们分析过，我们在事务中设置的`ActivityLifecycleItem`代表了`Activity`最终需要到达执行的生命周期，而中间的那些过渡生命周期就由`cycleToPath`方法推进执行，我们目前的生命周期状态为`ON_PAUSE`，而我们的目标生命周期为`ON_DESTROY`，中间还夹着一个`ON_STOP`，所以这个方法会帮我们执行`ClientTransactionHandler.handleStopActivity`方法，也就是`ActivityThread.handleStopActivity`方法

# handleStopActivity

```java
public void handleStopActivity(IBinder token, int configChanges,
        PendingTransactionActions pendingActions, boolean finalStateRequest /* false */, String reason) {
    final ActivityClientRecord r = mActivities.get(token);
    r.activity.mConfigChangeFlags |= configChanges;

    final StopInfo stopInfo = new StopInfo();
    //执行onStop生命周期
    performStopActivityInner(r, stopInfo, true /* saveState */, finalStateRequest,
            reason);

    //更新可见性
    updateVisibility(r, false);

    // Make sure any pending writes are now committed.
    //确保所有全局任务都被处理完成
    if (!r.isPreHoneycomb()) {
        QueuedWork.waitToFinish();
    }

    //记录Stop信息（不过在后续销毁链路中似乎并没有被用到）
    stopInfo.setActivity(r);
    stopInfo.setState(r.state);
    stopInfo.setPersistentState(r.persistentState);
    pendingActions.setStopInfo(stopInfo);
    mSomeActivitiesChanged = true;
}
```

接下来的路径就和其他生命周期差不多了，大部分内容我都用注释标注了，大家顺着往下看就行了

```java
/**
 * Core implementation of stopping an activity.
 * @param r Target activity client record.
 * @param info Action that will report activity stop to server.
 * @param saveState Flag indicating whether the activity state should be saved.
 * @param finalStateRequest Flag indicating if this call is handling final lifecycle state
 *                          request for a transaction.
 * @param reason Reason for performing this operation.
 */
private void performStopActivityInner(ActivityClientRecord r, StopInfo info,
        boolean saveState, boolean finalStateRequest /* false */, String reason) {
    if (r != null) {
        ... //异常状态处理

        // One must first be paused before stopped...
        //如果没有被暂停则先执行pause生命周期
        performPauseActivityIfNeeded(r, reason);

        ... //设置描述（Activity.onCreateDescription）

        //回调Activity的onStop方法
        callActivityOnStop(r, saveState, reason);
    }
}
```

```java
/**
 * Calls {@link Activity#onStop()} and {@link Activity#onSaveInstanceState(Bundle)}, and updates
 * the client record's state.
 * All calls to stop an activity must be done through this method to make sure that
 * {@link Activity#onSaveInstanceState(Bundle)} is also executed in the same call.
 */
private void callActivityOnStop(ActivityClientRecord r, boolean saveState, String reason) {
    // Before P onSaveInstanceState was called before onStop, starting with P it's
    // called after. Before Honeycomb state was always saved before onPause.
    //这里shouldSaveState为true，因为activity.mFinished早在performPauseActivity的时候就被设为了true
    final boolean shouldSaveState = saveState && !r.activity.mFinished && r.state == null
            && !r.isPreHoneycomb();
    //targetSdkVersion为Android P (Android 9)之前
    final boolean isPreP = r.isPreP();
    if (shouldSaveState && isPreP) {
        callActivityOnSaveInstanceState(r);
    }

    try {
        //执行stop生命周期
        r.activity.performStop(r.mPreserveWindow, reason);
    } catch (SuperNotCalledException e) {
        throw e;
    } catch (Exception e) {
        ...
    }
    //设置生命周期状态
    r.setState(ON_STOP);

    if (shouldSaveState && !isPreP) {
        callActivityOnSaveInstanceState(r);
    }
}
```

```java
final void performStop(boolean preserveWindow, String reason) {
    mDoReportFullyDrawn = false;
    //Loader相关，详见https://developer.android.com/guide/components/loaders
    mFragments.doLoaderStop(mChangingConfigurations /*retain*/);

    // Disallow entering picture-in-picture after the activity has been stopped
    //stop后禁用画中画
    mCanEnterPictureInPicture = false;

    if (!mStopped) {
        //分发PreStopped事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPreStopped回调
        dispatchActivityPreStopped();
        //关闭所有子窗口
        if (mWindow != null) {
            mWindow.closeAllPanels();
        }

        // If we're preserving the window, don't setStoppedState to true, since we
        // need the window started immediately again. Stopping the window will
        // destroys hardware resources and causes flicker.
        if (!preserveWindow && mToken != null && mParent == null) {
            //设置停止状态，释放硬件资源，销毁Surface
            WindowManagerGlobal.getInstance().setStoppedState(mToken, true);
        }

        //FragmentManager分发stop状态
        mFragments.dispatchStop();

        mCalled = false;
        //执行onStop回调
        mInstrumentation.callActivityOnStop(this);
        EventLogTags.writeWmOnStopCalled(mIdent, getComponentName().getClassName(), reason);
        if (!mCalled) {
            //必须要调用super.onStop方法
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onStop()");
        }

        //释放Cursors
        synchronized (mManagedCursors) {
            final int N = mManagedCursors.size();
            for (int i=0; i<N; i++) {
                ManagedCursor mc = mManagedCursors.get(i);
                if (!mc.mReleased) {
                    mc.mCursor.deactivate();
                    mc.mReleased = true;
                }
            }
        }

        mStopped = true;
        //分发PostStopped事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPostStopped回调
        dispatchActivityPostStopped();
    }
    mResumed = false;
}
```

通过`Instrumentation`执行`onStop`回调

```java
protected void onStop() {
    //ActionBar动画
    if (mActionBar != null) mActionBar.setShowHideAnimationEnabled(false);
    //共享元素动画
    mActivityTransitionState.onStop();
    //分发PostStopped事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPostStopped回调
    dispatchActivityStopped();
    mTranslucentCallback = null;
    //super.onStop标注为已被执行
    mCalled = true;

    ... //自动填充相关

    mEnterAnimationComplete = false;
}
```

这样，一整个`onStop`生命周期就执行完成了，最后还剩下个`onDestroy`，根据之前写的事务调度链路，现在应该走到了`DestroyActivityItem.execute`方法

```java
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    client.handleDestroyActivity(token, mFinished, mConfigChanges,
            false /* getNonConfigInstance */, "DestroyActivityItem");
}
```

可以看到，实际上就直接调用了`ActivityThread.handleDestroyActivity`方法

# handleDestroyActivity

```java
public void handleDestroyActivity(IBinder token, boolean finishing, int configChanges,
        boolean getNonConfigInstance, String reason) {
    //执行onDestroy生命周期
    ActivityClientRecord r = performDestroyActivity(token, finishing,
            configChanges, getNonConfigInstance, reason);
    if (r != null) {
        //清理之前设置的延时移除的window
        cleanUpPendingRemoveWindows(r, finishing);
        WindowManager wm = r.activity.getWindowManager();
        View v = r.activity.mDecor;
        if (v != null) {
            if (r.activity.mVisibleFromServer) {
                mNumVisibleActivities--;
            }
            IBinder wtoken = v.getWindowToken();
            if (r.activity.mWindowAdded) {
                if (r.mPreserveWindow) {
                    // Hold off on removing this until the new activity's
                    // window is being added.
                    r.mPendingRemoveWindow = r.window;
                    r.mPendingRemoveWindowManager = wm;
                    // We can only keep the part of the view hierarchy that we control,
                    // everything else must be removed, because it might not be able to
                    // behave properly when activity is relaunching.
                    //从DecorView中移除ContentView
                    r.window.clearContentView();
                } else {
                    //立刻执行View的移除操作，释放硬件资源，销毁Surface，回调View.onDetachedFromWindow
                    wm.removeViewImmediate(v);
                }
            }
            if (wtoken != null && r.mPendingRemoveWindow == null) {
                //移除指定Window下的所有rootView
                WindowManagerGlobal.getInstance().closeAll(wtoken,
                        r.activity.getClass().getName(), "Activity");
            } else if (r.mPendingRemoveWindow != null) {
                // We're preserving only one window, others should be closed so app views
                // will be detached before the final tear down. It should be done now because
                // some components (e.g. WebView) rely on detach callbacks to perform receiver
                // unregister and other cleanup.
                //移除指定Window下除了当前DecorView以外的所有rootView
                WindowManagerGlobal.getInstance().closeAllExceptView(token, v,
                        r.activity.getClass().getName(), "Activity");
            }
            r.activity.mDecor = null;
        }
        if (r.mPendingRemoveWindow == null) {
            // If we are delaying the removal of the activity window, then
            // we can't clean up all windows here.  Note that we can't do
            // so later either, which means any windows that aren't closed
            // by the app will leak.  Well we try to warning them a lot
            // about leaking windows, because that is a bug, so if they are
            // using this recreate facility then they get to live with leaks.
            WindowManagerGlobal.getInstance().closeAll(token,
                    r.activity.getClass().getName(), "Activity");
        }

        // Mocked out contexts won't be participating in the normal
        // process lifecycle, but if we're running with a proper
        // ApplicationContext we need to have it tear down things
        // cleanly.
        //清理Context
        Context c = r.activity.getBaseContext();
        if (c instanceof ContextImpl) {
            ((ContextImpl) c).scheduleFinalCleanup(
                    r.activity.getClass().getName(), "Activity");
        }
    }
    if (finishing) {
        try {
            //处理一些销毁后的事项，移除超时回调等
            ActivityTaskManager.getService().activityDestroyed(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
    mSomeActivitiesChanged = true;
}
```

`Window`这块的处理我目前也不太了解，等以后我学习了`WMS`那块再补吧

其他的和之前的套路一样，调用`performDestroyActivity`方法执行销毁`Activity`

```java
/** Core implementation of activity destroy call. */
ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
        int configChanges, boolean getNonConfigInstance, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    Class<? extends Activity> activityClass = null;
    if (r != null) {
        activityClass = r.activity.getClass();
        r.activity.mConfigChangeFlags |= configChanges;
        if (finishing) {
            r.activity.mFinished = true;
        }

        //如果没有被暂停则先执行pause生命周期
        performPauseActivityIfNeeded(r, "destroy");

        //如果没有被停职则先执行stop生命周期
        if (!r.stopped) {
            callActivityOnStop(r, false /* saveState */, "destroy");
        }
        if (getNonConfigInstance) {
            ... //getNonConfigInstance为false，这里不执行
        }
        try {
            r.activity.mCalled = false;
            //执行onDestroy回调
            mInstrumentation.callActivityOnDestroy(r.activity);
            //必须要调用super.onDestroy方法
            if (!r.activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + safeToComponentShortString(r.intent) +
                    " did not call through to super.onDestroy()");
            }
            //关闭所有子窗口
            if (r.window != null) {
                r.window.closeAllPanels();
            }
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            ...
        }
        //设置生命周期状态
        r.setState(ON_DESTROY);
    }
    //空闲时清理资源
    schedulePurgeIdler();
    // updatePendingActivityConfiguration() reads from mActivities to update
    // ActivityClientRecord which runs in a different thread. Protect modifications to
    // mActivities to avoid race.
    synchronized (mResourcesManager) {
        mActivities.remove(token);
    }
    //严格模式更新Activity计数器，与实际Activity数量对比，判断是否产生内存泄漏
    StrictMode.decrementExpectedActivityCount(activityClass);
    return r;
}
```

通过`Instrumentation`调用`Activity.performDestroy`方法

```java
final void performDestroy() {
    //分发PreDestroyed事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPreDestroyed回调
    dispatchActivityPreDestroyed();
    mDestroyed = true;
    mWindow.destroy();
    mFragments.dispatchDestroy();
    onDestroy();
    EventLogTags.writeWmOnDestroyCalled(mIdent, getComponentName().getClassName(),
            "performDestroy");
    mFragments.doLoaderDestroy();
    if (mVoiceInteractor != null) {
        mVoiceInteractor.detachActivity();
    }
    //分发PostDestroyed事件，执行所有注册的ActivityLifecycleCallbacks的onActivityPostDestroyed回调
    dispatchActivityPostDestroyed();
}
```

```java
protected void onDestroy() {
    //super.onDestroy标注为已被执行
    mCalled = true;

    ... //自动填充相关

    // dismiss any dialogs we are managing.
    //关闭所有被管理的Dialog
    if (mManagedDialogs != null) {
        final int numDialogs = mManagedDialogs.size();
        for (int i = 0; i < numDialogs; i++) {
            final ManagedDialog md = mManagedDialogs.valueAt(i);
            if (md.mDialog.isShowing()) {
                md.mDialog.dismiss();
            }
        }
        mManagedDialogs = null;
    }

    // close any cursors we are managing.
    //关闭所有被管理的Cursor
    synchronized (mManagedCursors) {
        int numCursors = mManagedCursors.size();
        for (int i = 0; i < numCursors; i++) {
            ManagedCursor c = mManagedCursors.get(i);
            if (c != null) {
                c.mCursor.close();
            }
        }
        mManagedCursors.clear();
    }

    // Close any open search dialog
    //关闭系统搜索服务的弹窗
    if (mSearchManager != null) {
        mSearchManager.stopSearch();
    }

    if (mActionBar != null) {
        mActionBar.onDestroy();
    }

    //分发Destroyed事件，执行所有注册的ActivityLifecycleCallbacks的onActivityDestroyed回调
    dispatchActivityDestroyed();

    //内容捕获服务
    notifyContentCaptureManagerIfNeeded(CONTENT_CAPTURE_STOP);
}
```

`DestroyActivityItem`没有重写`postExecute`方法，所以到此为止，`Activity`整个销毁流程就结束了

# Tips

我们通过本篇文章的分析，可以发现，触发`Activity`销毁后，`onStop`和`onDestroy`这两个生命周期回调的触发时机是不确定的，如果有需求需要在确定`Activity`要被销毁后立刻执行，我们可以在`onPause`回调中调用`Activity.isFinishing`方法判断`mFinished`标志是否被置`true`，如果为`true`则可以判定这个`Activity`将被销毁

# 结尾

至此，`Activity`的启动流程和销毁流程我们都分析完了，后面应该暂时不会再写`Activity`相关的源码分析了

之后的一段时间，我可能会将我的精力投入到`AIGC`的技术调研中，`Android源码分析`这一系列的后续更新可能会放慢，希望大家多多谅解