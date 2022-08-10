---
title: Android源码分析 - Activity启动流程（上）
date: 2022-08-01 15:19:35
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

作为一名`Android`开发，我们最熟悉并且最常打交道的当然非四大组件中的`Activity`莫属，这次我们就来讲讲一个`Activity`是怎样启动起来的

本来本篇想要讲`ActivityManagerService`的，但`AMS`中的内容过多过于繁杂，不如用这种以线及面的方式，通过`Activity`的启动流程这一条线，去了解`ActivityThread`，`AMS`等是怎么工作的

# Android窗口结构关系

在开始正式深入代码之前，我们需要先理一下`Android`的窗口结构关系，以免后面看到源码里各种`parent`、`child`头晕，我画了一张树状关系图来描述它

![Android窗口结构关系](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20-%20Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%EF%BC%88%E4%B8%8A%EF%BC%89-Android%E7%AA%97%E5%8F%A3%E7%BB%93%E6%9E%84%E5%85%B3%E7%B3%BB.png)

上图中的所有类都是`WindowContainer`的子类，`WindowContainer`是一个窗口容器，它的`child`也是`WindowContainer`，它是用来管理窗口子容器的

可以先不用纠结理解这张图中的关系，顺着源码往下看，碰到不理解的地方回头看一下就可以了

# startActivity

作为`Android`开发，`startActivity`这个方法一定非常熟悉，我们以这个函数作为入口来分析`Activity`的启动流程

`Activity`和`ContextImpl`的`startActivity`方法实现不太一样，但最终都调用了`Instrumentation.execStartActivity`方法

# Instrumentation

路径：`frameworks/base/core/java/android/app/Instrumentation.java`

以下是Google官方对这个类功能的注释

*Base class for implementing application instrumentation code.  When running with instrumentation turned on, this class will be instantiated for you before any of the application code, allowing you to monitor all of the interaction the system has with the application.  An Instrumentation implementation is described to the system through an AndroidManifest.xml's \<instrumentation\> tag.*

简单翻译一下，就是这个类是用于监控系统与应用的交互的（`onCreate`等生命周期会经历`Instrumentation`这么一环），它会在任何`App`代码执行前被初始化。

本人猜测，这个类主要存在的意义是为了给自动化测试提供一个切入点

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    //记录调用者
    Uri referrer = target != null ? target.onProvideReferrer() : null;
    if (referrer != null) {
        intent.putExtra(Intent.EXTRA_REFERRER, referrer);
    }

    ... //自动化测试相关

    try {
        //迁移额外的URI数据流到剪贴板（处理Intent使用ACTION_SEND或ACTION_SEND_MULTIPLE共享数据的情况）
        intent.migrateExtraStreamToClipData(who);
        //处理离开当前App进程的情况
        intent.prepareToLeaveProcess(who);
        //请求ATMS启动Activity
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getBasePackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
        //检查异常情况，抛出对应异常
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

这个函数调用了`ActivityTaskManager.getService.startActivity`方法

```java
public static IActivityTaskManager getService() {
    return IActivityTaskManagerSingleton.get();
}

private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
            @Override
            protected IActivityTaskManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                return IActivityTaskManager.Stub.asInterface(b);
            }
        };
```

这里`getService`出来的很明显的是一个远程`binder`对象，我们之前已经分析过那么多`binder`知识了，以后就不再过多啰嗦了，这里实际上调用的是`ActivityTaskManagerSerivce`（以下简称`ATMS`）的`startActivity`方法

# ActivityTaskManagerSerivce

`ATMS`是`Android 10`以后新加的一个服务，用来专门处理`Activity`相关工作，分担`AMS`的工作

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

@Override
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}

private int startActivityAsUser(IApplicationThread caller, String callingPackage,
        @Nullable String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
    //断言发起startActivity请求方的UID和callingPackage指向的是同一个App
    assertPackageMatchesCallingUid(callingPackage);
    //确认请求方没有被隔离
    enforceNotIsolatedCaller("startActivityAsUser");

    //检查并获取当前用户ID（多用户模式）
    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    //使用ActivityStarter启动Activity
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller) //调用方ApplicationThread
            .setCallingPackage(callingPackage) //调用方包名
            .setCallingFeatureId(callingFeatureId) // Context.getAttributionTag()
            .setResolvedType(resolvedType) //设置Intent解析类型
            .setResultTo(resultTo) //设置目标Activity Token（ContextImpl.startActivity传入参数为null）
            .setResultWho(resultWho) //设置目标Activity（ContextImpl.startActivity传入参数为null）
            .setRequestCode(requestCode) //设置requestCode
            .setStartFlags(startFlags) // startFlags == 0
            .setProfilerInfo(profilerInfo) // null
            .setActivityOptions(bOptions) //设置Activity Options Bundle
            .setUserId(userId) //设置用户ID
            .execute();

}
```

这个函数大部分内容都是检查，最重要的是最后一段使用`ActivityStarter`启动`Activity`，首先通过`ActivityStartController`的`obtainStarter`方法获取一个`ActivityStarter`实例，然后调用各种set方法设置参数，最后执行`execute`方法执行

# ActivityStarter

这个类从名字就能看出来，就是一个专门处理`Activity`启动的类

```java
int execute() {
    try {
        // Refuse possible leaked file descriptors
        //校验Intent，不允许其携带fd
        if (mRequest.intent != null && mRequest.intent.hasFileDescriptors()) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        final LaunchingState launchingState;
        synchronized (mService.mGlobalLock) {
            //通过Token获取调用方ActivityRecord
            final ActivityRecord caller = ActivityRecord.forTokenLocked(mRequest.resultTo);
            //记录启动状态
            launchingState = mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(
                    mRequest.intent, caller);
        }

        // If the caller hasn't already resolved the activity, we're willing
        // to do so here. If the caller is already holding the WM lock here,
        // and we need to check dynamic Uri permissions, then we're forced
        // to assume those permissions are denied to avoid deadlocking.
        //通过Intent解析Activity信息
        if (mRequest.activityInfo == null) {
            mRequest.resolveActivity(mSupervisor);
        }

        int res;
        synchronized (mService.mGlobalLock) {
            ... //处理Configuration

            //清除Binder调用方UID和PID，用当前进程的UID和PID替代，并返回之前的UID和PID（UID：前32位，PID：后32位）
            final long origId = Binder.clearCallingIdentity();

            //解析成为重量级进程（如果设置了相关flag的话）
            //这里的重量级进程指的是不能保存状态的应用进程
            res = resolveToHeavyWeightSwitcherIfNeeded();
            if (res != START_SUCCESS) {
                return res;
            }
            //执行请求
            res = executeRequest(mRequest);

            //恢复之前的Binder调用方UID和PID
            Binder.restoreCallingIdentity(origId);

            ... //更新Configuration

            // Notify ActivityMetricsLogger that the activity has launched.
            // ActivityMetricsLogger will then wait for the windows to be drawn and populate
            // WaitResult.
            //记录启动状态
            mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(launchingState, res,
                    mLastStartActivityRecord);
            //返回启动结果
            return getExternalResult(mRequest.waitResult == null ? res
                    : waitForResult(res, mLastStartActivityRecord));
        }
    } finally {
        //清理回收工作
        onExecutionComplete();
    }
}
```

这个函数每一步做了什么我都用注释标出来了，大家看看就好，重点在于其中的`executeRequest(mRequest)`

```java
/**
    * Executing activity start request and starts the journey of starting an activity. Here
    * begins with performing several preliminary checks. The normally activity launch flow will
    * go through {@link #startActivityUnchecked} to {@link #startActivityInner}.
    */
private int executeRequest(Request request) {
    if (TextUtils.isEmpty(request.reason)) {
        throw new IllegalArgumentException("Need to specify a reason.");
    }
    mLastStartReason = request.reason;
    mLastStartActivityTimeMs = System.currentTimeMillis();
    mLastStartActivityRecord = null;

    final IApplicationThread caller = request.caller;
    Intent intent = request.intent;
    NeededUriGrants intentGrants = request.intentGrants;
    String resolvedType = request.resolvedType;
    ActivityInfo aInfo = request.activityInfo;
    ResolveInfo rInfo = request.resolveInfo;
    final IVoiceInteractionSession voiceSession = request.voiceSession;
    final IBinder resultTo = request.resultTo;
    String resultWho = request.resultWho;
    int requestCode = request.requestCode;
    int callingPid = request.callingPid;
    int callingUid = request.callingUid;
    String callingPackage = request.callingPackage;
    String callingFeatureId = request.callingFeatureId;
    final int realCallingPid = request.realCallingPid;
    final int realCallingUid = request.realCallingUid;
    final int startFlags = request.startFlags;
    final SafeActivityOptions options = request.activityOptions;
    Task inTask = request.inTask;

    int err = ActivityManager.START_SUCCESS;
    // Pull the optional Ephemeral Installer-only bundle out of the options early.
    final Bundle verificationBundle =
            options != null ? options.popAppVerificationBundle() : null;

    WindowProcessController callerApp = null;
    if (caller != null) {
        //获取调用方应用进程对应的WindowProcessController
        //这个类是用于和ProcessRecord进行通讯的
        callerApp = mService.getProcessController(caller);
        if (callerApp != null) {
            callingPid = callerApp.getPid();
            callingUid = callerApp.mInfo.uid;
        } else {
            //异常情况，startActivity的调用方进程不存在或未启动
            Slog.w(TAG, "Unable to find app for caller " + caller + " (pid=" + callingPid
                    + ") when starting: " + intent.toString());
            err = ActivityManager.START_PERMISSION_DENIED;
        }
    }

    //获取当前用户ID
    final int userId = aInfo != null && aInfo.applicationInfo != null
            ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;
    if (err == ActivityManager.START_SUCCESS) {
        Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                + "} from uid " + callingUid);
    }

    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    //调用方Activity Token != null
    if (resultTo != null) {
        //获取调用方ActivityRecord（要求存在任意一个Window栈中，即是RootWindow的子嗣）
        sourceRecord = mRootWindowContainer.isInAnyStack(resultTo);
        if (DEBUG_RESULTS) {
            Slog.v(TAG_RESULTS, "Will send result to " + resultTo + " " + sourceRecord);
        }
        if (sourceRecord != null) {
            //调用方需要response
            if (requestCode >= 0 && !sourceRecord.finishing) {
                resultRecord = sourceRecord;
            }
        }
    }

    final int launchFlags = intent.getFlags();
    //多Activity传值场景
    if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
        ...
    }

    //找不到可以处理此Intent的组件
    if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
        // We couldn't find a class that can handle the given Intent.
        // That's the end of that!
        err = ActivityManager.START_INTENT_NOT_RESOLVED;
    }

    //Intent中解析不出相应的Activity信息
    if (err == ActivityManager.START_SUCCESS && aInfo == null) {
        // We couldn't find the specific class specified in the Intent.
        // Also the end of the line.
        err = ActivityManager.START_CLASS_NOT_FOUND;
    }

    if (err == ActivityManager.START_SUCCESS && sourceRecord != null
            && sourceRecord.getTask().voiceSession != null) {
        ... //语言交互相关
    }

    if (err == ActivityManager.START_SUCCESS && voiceSession != null) {
        ... //语言交互相关
    }

    final ActivityStack resultStack = resultRecord == null
            ? null : resultRecord.getRootTask();

    if (err != START_SUCCESS) {
        //回调给调用方Activity结果
        if (resultRecord != null) {
            resultRecord.sendResult(INVALID_UID, resultWho, requestCode, RESULT_CANCELED,
                    null /* data */, null /* dataGrants */);
        }
        SafeActivityOptions.abort(options);
        return err;
    }

    //检查启动Activity的权限
    boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
            requestCode, callingPid, callingUid, callingPackage, callingFeatureId,
            request.ignoreTargetSecurity, inTask != null, callerApp, resultRecord, resultStack);
    abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
            callingPid, resolvedType, aInfo.applicationInfo);
    abort |= !mService.getPermissionPolicyInternal().checkStartActivity(intent, callingUid,
            callingPackage);

    boolean restrictedBgActivity = false;
    if (!abort) {
        try {
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER,
                    "shouldAbortBackgroundActivityStart");
            //检查是否要限制后台启动Activity
            restrictedBgActivity = shouldAbortBackgroundActivityStart(callingUid,
                    callingPid, callingPackage, realCallingUid, realCallingPid, callerApp,
                    request.originatingPendingIntent, request.allowBackgroundActivityStart,
                    intent);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
        }
    }

    // Merge the two options bundles, while realCallerOptions takes precedence.
    //过渡动画相关
    ActivityOptions checkedOptions = options != null
            ? options.getOptions(intent, aInfo, callerApp, mSupervisor) : null;
    if (request.allowPendingRemoteAnimationRegistryLookup) {
        checkedOptions = mService.getActivityStartController()
                .getPendingRemoteAnimationRegistry()
                .overrideOptionsIfNeeded(callingPackage, checkedOptions);
    }
    if (mService.mController != null) {
        try {
            // The Intent we give to the watcher has the extra data stripped off, since it
            // can contain private information.
            Intent watchIntent = intent.cloneFilter();
            //这个方法似乎只打印了一些日志，恒返回true，即abort |= false
            abort |= !mService.mController.activityStarting(watchIntent,
                    aInfo.applicationInfo.packageName);
        } catch (RemoteException e) {
            mService.mController = null;
        }
    }

    //初始化ActivityStartInterceptor
    mInterceptor.setStates(userId, realCallingPid, realCallingUid, startFlags, callingPackage,
            callingFeatureId);
    if (mInterceptor.intercept(intent, rInfo, aInfo, resolvedType, inTask, callingPid,
            callingUid, checkedOptions)) {
        // activity start was intercepted, e.g. because the target user is currently in quiet
        // mode (turn off work) or the target application is suspended
        //拦截并转化成其他的启动模式
        intent = mInterceptor.mIntent;
        rInfo = mInterceptor.mRInfo;
        aInfo = mInterceptor.mAInfo;
        resolvedType = mInterceptor.mResolvedType;
        inTask = mInterceptor.mInTask;
        callingPid = mInterceptor.mCallingPid;
        callingUid = mInterceptor.mCallingUid;
        checkedOptions = mInterceptor.mActivityOptions;

        // The interception target shouldn't get any permission grants
        // intended for the original destination
        intentGrants = null;
    }

    if (abort) {
        //回调给调用方Activity结果
        if (resultRecord != null) {
            resultRecord.sendResult(INVALID_UID, resultWho, requestCode, RESULT_CANCELED,
                    null /* data */, null /* dataGrants */);
        }
        // We pretend to the caller that it was really started, but they will just get a
        // cancel result.
        ActivityOptions.abort(checkedOptions);
        return START_ABORTED;
    }

    // If permissions need a review before any of the app components can run, we
    // launch the review activity and pass a pending intent to start the activity
    // we are to launching now after the review is completed.
    if (aInfo != null) {
        //如果启动的Activity没有相应权限，则需要用户手动确认允许权限后，再进行启动工作
        if (mService.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                aInfo.packageName, userId)) {
            //将原来的Intent包装在新的Intent中，用这个确认权限的新Intent继续后面的启动工作
            final IIntentSender target = mService.getIntentSenderLocked(
                    ActivityManager.INTENT_SENDER_ACTIVITY, callingPackage, callingFeatureId,
                    callingUid, userId, null, null, 0, new Intent[]{intent},
                    new String[]{resolvedType}, PendingIntent.FLAG_CANCEL_CURRENT
                            | PendingIntent.FLAG_ONE_SHOT, null);

            Intent newIntent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);

            int flags = intent.getFlags();
            flags |= Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS;

            /*
                * Prevent reuse of review activity: Each app needs their own review activity. By
                * default activities launched with NEW_TASK or NEW_DOCUMENT try to reuse activities
                * with the same launch parameters (extras are ignored). Hence to avoid possible
                * reuse force a new activity via the MULTIPLE_TASK flag.
                *
                * Activities that are not launched with NEW_TASK or NEW_DOCUMENT are not re-used,
                * hence no need to add the flag in this case.
                */
            if ((flags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_NEW_DOCUMENT)) != 0) {
                flags |= Intent.FLAG_ACTIVITY_MULTIPLE_TASK;
            }
            newIntent.setFlags(flags);

            newIntent.putExtra(Intent.EXTRA_PACKAGE_NAME, aInfo.packageName);
            newIntent.putExtra(Intent.EXTRA_INTENT, new IntentSender(target));
            if (resultRecord != null) {
                newIntent.putExtra(Intent.EXTRA_RESULT_NEEDED, true);
            }
            intent = newIntent;

            // The permissions review target shouldn't get any permission
            // grants intended for the original destination
            intentGrants = null;

            resolvedType = null;
            callingUid = realCallingUid;
            callingPid = realCallingPid;

            rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId, 0,
                    computeResolveFilterUid(
                            callingUid, realCallingUid, request.filterCallingUid));
            aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags,
                    null /*profilerInfo*/);
        }
    }

    // If we have an ephemeral app, abort the process of launching the resolved intent.
    // Instead, launch the ephemeral installer. Once the installer is finished, it
    // starts either the intent we resolved here [on install error] or the ephemeral
    // app [on install success].
    if (rInfo != null && rInfo.auxiliaryInfo != null) {
        ... //Instant App相关
    }

    //创建启动Activity的ActivityRecord
    final ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
            callingPackage, callingFeatureId, intent, resolvedType, aInfo,
            mService.getGlobalConfiguration(), resultRecord, resultWho, requestCode,
            request.componentSpecified, voiceSession != null, mSupervisor, checkedOptions,
            sourceRecord);
    mLastStartActivityRecord = r;

    if (r.appTimeTracker == null && sourceRecord != null) {
        // If the caller didn't specify an explicit time tracker, we want to continue
        // tracking under any it has.
        r.appTimeTracker = sourceRecord.appTimeTracker;
    }

    //获取顶层焦点的Acticity栈
    final ActivityStack stack = mRootWindowContainer.getTopDisplayFocusedStack();

    // If we are starting an activity that is not from the same uid as the currently resumed
    // one, check whether app switches are allowed.
    //当此时栈顶Activity UID != 调用方 UID的时候（比如悬浮窗）
    if (voiceSession == null && stack != null && (stack.getResumedActivity() == null
            || stack.getResumedActivity().info.applicationInfo.uid != realCallingUid)) {
        //检查是否可以直接切换应用
        // 1. 设置的 mAppSwitchesAllowedTime < 当前系统时间（stopAppSwitches）
        // 2. 调用方在最近任务中
        // 3. 调用方具有 STOP_APP_SWITCHES 权限
        // ...
        if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                realCallingPid, realCallingUid, "Activity start")) {
            //加入到延时启动列表中
            if (!(restrictedBgActivity && handleBackgroundActivityAbort(r))) {
                mController.addPendingActivityLaunch(new PendingActivityLaunch(r,
                        sourceRecord, startFlags, stack, callerApp, intentGrants));
            }
            ActivityOptions.abort(checkedOptions);
            return ActivityManager.START_SWITCHES_CANCELED;
        }
    }

    //回调处理延迟应用切换
    mService.onStartActivitySetDidAppSwitch();
    mController.doPendingActivityLaunches(false);

    //核心：进入Activity启动的下一阶段
    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
            request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
            restrictedBgActivity, intentGrants);

    if (request.outActivity != null) {
        request.outActivity[0] = mLastStartActivityRecord;
    }

    return mLastStartActivityResult;
}
```

这个函数大部分都是检查工作，这些可以看我标的注释，基本上介绍的比较详细了，然后进入到Activity启动的下一步，`startActivityUnchecked`

```java
/**
    * Start an activity while most of preliminary checks has been done and caller has been
    * confirmed that holds necessary permissions to do so.
    * Here also ensures that the starting activity is removed if the start wasn't successful.
    */
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    int result = START_CANCELED;
    final ActivityStack startedActivityStack;
    try {
        //暂停布局工作，避免重复刷新
        mService.deferWindowLayout();
        Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
        //接着把启动Activity工作交给这个方法
        result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
        //进行一些更新Configuration，清理栈等收尾工作
        startedActivityStack = handleStartResult(r, result);
        //恢复布局工作
        mService.continueWindowLayout();
    }

    postStartActivityProcessing(r, result, startedActivityStack);

    return result;
}
```

这个方法也不是主要逻辑所在，我们往下接着看`startActivityInner`方法

```java
/**
    * Start an activity and determine if the activity should be adding to the top of an existing
    * task or delivered new intent to an existing activity. Also manipulating the activity task
    * onto requested or valid stack/display.
    *
    * Note: This method should only be called from {@link #startActivityUnchecked}.
    */

// TODO(b/152429287): Make it easier to exercise code paths through startActivityInner
@VisibleForTesting
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, Task inTask,
        boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    //设置初始化参数
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor, restrictedBgActivity);
    //计算处理Activity启动模式
    computeLaunchingTaskFlags();
    //计算调用方Activity任务栈
    computeSourceStack();

    //将flags设置为调整后的LaunchFlags
    mIntent.setFlags(mLaunchFlags);

    //查找是否有可复用的Task
    final Task reusedTask = getReusableTask();

    // If requested, freeze the task list
    if (mOptions != null && mOptions.freezeRecentTasksReordering()
            && mSupervisor.mRecentTasks.isCallerRecents(r.launchedFromUid)
            && !mSupervisor.mRecentTasks.isFreezeTaskListReorderingSet()) {
        mFrozeTaskList = true;
        mSupervisor.mRecentTasks.setFreezeTaskListReordering();
    }

    // Compute if there is an existing task that should be used for.
    //计算是否存在可使用的现有Task
    final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
    final boolean newTask = targetTask == null;
    mTargetTask = targetTask;

    //计算启动参数
    computeLaunchParams(r, sourceRecord, targetTask);

    // Check if starting activity on given task or on a new task is allowed.
    //检查是否允许在targetTask上或者新建Task启动
    int startResult = isAllowedToStart(r, newTask, targetTask);
    if (startResult != START_SUCCESS) {
        return startResult;
    }

    //获得栈顶未finish的ActivityRecord
    final ActivityRecord targetTaskTop = newTask
            ? null : targetTask.getTopNonFinishingActivity();
    if (targetTaskTop != null) {
        // Recycle the target task for this launch.
        //回收，准备复用这个Task
        startResult = recycleTask(targetTask, targetTaskTop, reusedTask, intentGrants);
        if (startResult != START_SUCCESS) {
            return startResult;
        }
    } else {
        mAddingToTask = true;
    }

    // If the activity being launched is the same as the one currently at the top, then
    // we need to check if it should only be launched once.
    //处理singleTop启动模式
    final ActivityStack topStack = mRootWindowContainer.getTopDisplayFocusedStack();
    if (topStack != null) {
        startResult = deliverToCurrentTopIfNeeded(topStack, intentGrants);
        if (startResult != START_SUCCESS) {
            return startResult;
        }
    }

    //复用或创建Activity栈
    if (mTargetStack == null) {
        mTargetStack = getLaunchStack(mStartActivity, mLaunchFlags, targetTask, mOptions);
    }

    if (newTask) { //开启新Task
        final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTask() : null;
        //复用或新建一个Task，并建立Task与ActivityRecord之间的关联
        setNewTask(taskToAffiliate);
        if (mService.getLockTaskController().isLockTaskModeViolation(
                mStartActivity.getTask())) {
            Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
            return START_RETURN_LOCK_TASK_MODE_VIOLATION;
        }
    } else if (mAddingToTask) { //复用Task
        //将启动Activity添加到targetTask容器顶部或将其父容器替换成targetTask（也会将启动Activity添加到targetTask容器顶部）
        addOrReparentStartingActivity(targetTask, "adding to task");
    }

    if (!mAvoidMoveToFront && mDoResume) {
        mTargetStack.getStack().moveToFront("reuseOrNewTask", targetTask);
        if (mOptions != null) {
            if (mOptions.getTaskAlwaysOnTop()) {
                mTargetStack.setAlwaysOnTop(true);
            }
        }
        if (!mTargetStack.isTopStackInDisplayArea() && mService.mInternal.isDreaming()) {
            // Launching underneath dream activity (fullscreen, always-on-top). Run the launch-
            // -behind transition so the Activity gets created and starts in visible state.
            mLaunchTaskBehind = true;
            r.mLaunchTaskBehind = true;
        }
    }

    ...

    mTargetStack.mLastPausedActivity = null;

    mRootWindowContainer.sendPowerHintForLaunchStartIfNeeded(
            false /* forceSend */, mStartActivity);

    //将Task移到ActivityStack容器顶部
    mTargetStack.startActivityLocked(mStartActivity, topStack.getTopNonFinishingActivity(),
            newTask, mKeepCurTransition, mOptions);
    if (mDoResume) {
        final ActivityRecord topTaskActivity =
                mStartActivity.getTask().topRunningActivityLocked();
        //启动的Activity不可获得焦点，无法恢复它
        if (!mTargetStack.isTopActivityFocusable()
                || (topTaskActivity != null && topTaskActivity.isTaskOverlay()
                && mStartActivity != topTaskActivity)) {
            // If the activity is not focusable, we can't resume it, but still would like to
            // make sure it becomes visible as it starts (this will also trigger entry
            // animation). An example of this are PIP activities.
            // Also, we don't want to resume activities in a task that currently has an overlay
            // as the starting activity just needs to be in the visible paused state until the
            // over is removed.
            // Passing {@code null} as the start parameter ensures all activities are made
            // visible.
            mTargetStack.ensureActivitiesVisible(null /* starting */,
                    0 /* configChanges */, !PRESERVE_WINDOWS);
            // Go ahead and tell window manager to execute app transition for this activity
            // since the app transition will not be triggered through the resume channel.
            mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
        } else {
            // If the target stack was not previously focusable (previous top running activity
            // on that stack was not visible) then any prior calls to move the stack to the
            // will not update the focused stack.  If starting the new activity now allows the
            // task stack to be focusable, then ensure that we now update the focused stack
            // accordingly.
            if (mTargetStack.isTopActivityFocusable()
                    && !mRootWindowContainer.isTopDisplayFocusedStack(mTargetStack)) {
                mTargetStack.moveToFront("startActivityInner");
            }
            //重点：恢复栈顶Activities
            mRootWindowContainer.resumeFocusedStacksTopActivities(
                    mTargetStack, mStartActivity, mOptions);
        }
    }
    mRootWindowContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

    // Update the recent tasks list immediately when the activity starts
    //当Activity启动后立刻更新最近任务列表
    mSupervisor.mRecentTasks.add(mStartActivity.getTask());
    mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(),
            mPreferredWindowingMode, mPreferredTaskDisplayArea, mTargetStack);

    return START_SUCCESS;
}
```

这里主要做了一些`Task`和栈的操作，是否可以复用栈，是否需要新栈，处理栈顶复用等相关操作。我们需要注意一下这里关于`Task`的操作，不管是新建`Task`（`newTask`）还是复用`Task`（`mAddingToTask`），都会调用到`addOrReparentStartingActivity`方法将启动`ActivityRecord`添加到`targetTask`容器顶部（`newTask`的情况下会调用`setNewTask`方法先复用或创建`Task`，然后再用这个`Task`调用`addOrReparentStartingActivity`方法），之后调用`mTargetStack.startActivityLocked`方法将`Task`移到`mTargetStack`容器顶部，此时调用`mTargetStack.topRunningActivity`便会得到我们将要启动的这个`ActivityRecord`

最后判断目标`Activity`是否可获得焦点，当可获得焦点的时候，调用`RootWindowContainer.resumeFocusedStacksTopActivities`方法恢复`Activity`

要注意，从这个方法开始的以后的方法不再只是针对`Activity`启动的方法，它们有可能被多方调用，所以其中的一些步骤case我们是不会经历的，可以忽略掉这部分

# RootWindowContainer

`RootWindowContainer`是显示窗口的根窗口容器，它主要是用来管理显示屏幕的

```java
boolean resumeFocusedStacksTopActivities(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

    if (!mStackSupervisor.readyToResume()) {
        return false;
    }

    boolean result = false;
    //目标栈在栈顶显示区域
    if (targetStack != null && (targetStack.isTopStackInDisplayArea()
            || getTopDisplayFocusedStack() == targetStack)) {
        //使用目标栈进行恢复
        result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }

    //可能存在多显示设备（投屏等）
    for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
        boolean resumedOnDisplay = false;
        final DisplayContent display = getChildAt(displayNdx);
        for (int tdaNdx = display.getTaskDisplayAreaCount() - 1; tdaNdx >= 0; --tdaNdx) {
            final TaskDisplayArea taskDisplayArea = display.getTaskDisplayAreaAt(tdaNdx);
            for (int sNdx = taskDisplayArea.getStackCount() - 1; sNdx >= 0; --sNdx) {
                final ActivityStack stack = taskDisplayArea.getStackAt(sNdx);
                final ActivityRecord topRunningActivity = stack.topRunningActivity();
                if (!stack.isFocusableAndVisible() || topRunningActivity == null) {
                    continue;
                }
                if (stack == targetStack) {
                    //如果进入到这里，代表着targetStack在上面已经恢复过了，此时只需要记录结果即可
                    resumedOnDisplay |= result;
                    continue;
                }
                if (taskDisplayArea.isTopStack(stack) && topRunningActivity.isState(RESUMED)) {
                    //执行切换效果
                    stack.executeAppTransition(targetOptions);
                } else {
                    //使顶部显示的Activity执行Resume、Pause或Start生命周期
                    resumedOnDisplay |= topRunningActivity.makeActiveIfNeeded(target);
                }
            }
        }
        if (!resumedOnDisplay) {
            // In cases when there are no valid activities (e.g. device just booted or launcher
            // crashed) it's possible that nothing was resumed on a display. Requesting resume
            // of top activity in focused stack explicitly will make sure that at least home
            // activity is started and resumed, and no recursion occurs.
            //当没有任何有效的Activity的时候（设备刚启动或Launcher崩溃），可能没有任何东西可被恢复
            //这时候使用DisplayContent中的焦点栈进行恢复
            //如果连存在焦点的栈都没有，则恢复Launcher的Activity
            final ActivityStack focusedStack = display.getFocusedStack();
            if (focusedStack != null) {
                result |= focusedStack.resumeTopActivityUncheckedLocked(target, targetOptions);
            } else if (targetStack == null) {
                result |= resumeHomeActivity(null /* prev */, "no-focusable-task",
                        display.getDefaultTaskDisplayArea());
            }
        }
    }

    return result;
}
```

这个方法主要做了几件事：

- 如果目标栈在栈顶显示区域，执行Resume

- 遍历显示设备，从中遍历所有有焦点并且可见的栈，对其栈顶`Activity`执行相应的切换效果及生命周期

- 对每个显示设备，如果存在焦点栈，则使用其执行Resume，否则启动`Launcher`

在正常情况下，我们会走进`ActivityStack.resumeTopActivityUncheckedLocked`这个方法

# ActivityStack

`Activity`栈，用于管理栈中的`Activity`

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    if (mInResumeTopActivity) {
        // Don't even start recursing.
        //防止递归
        return false;
    }

    boolean result = false;
    try {
        // Protect against recursion.
        //防止递归
        mInResumeTopActivity = true;
        result = resumeTopActivityInnerLocked(prev, options);

        // When resuming the top activity, it may be necessary to pause the top activity (for
        // example, returning to the lock screen. We suppress the normal pause logic in
        // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
        // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
        // to ensure any necessary pause logic occurs. In the case where the Activity will be
        // shown regardless of the lock screen, the call to
        // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
        final ActivityRecord next = topRunningActivity(true /* focusableOnly */);
        if (next == null || !next.canTurnScreenOn()) {
            //准备休眠
            checkReadyForSleep();
        }
    } finally {
        mInResumeTopActivity = false;
    }

    return result;
}
```

这里做了一个防止递归调用的措施，接下来调用了`resumeTopActivityInnerLocked`方法

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    if (!mAtmService.isBooting() && !mAtmService.isBooted()) {
        // Not ready yet!
        //ATMS服务尚未准备好
        return false;
    }

    // Find the next top-most activity to resume in this stack that is not finishing and is
    // focusable. If it is not focusable, we will fall into the case below to resume the
    // top activity in the next focusable task.
    //在之前我们已经把要启动的ActivityRecord加到了栈顶
    ActivityRecord next = topRunningActivity(true /* focusableOnly */);

    final boolean hasRunningActivity = next != null;

    ...

    if (!hasRunningActivity) {
        // There are no activities left in the stack, let's look somewhere else.
        return resumeNextFocusableActivityWhenStackIsEmpty(prev, options);
    }

    next.delayedResume = false;
    final TaskDisplayArea taskDisplayArea = getDisplayArea();

    // If the top activity is the resumed one, nothing to do.
    //如果需要Resume的已在顶部且状态为Resume，不需要做任何事
    //启动Activity不会碰到这种case
    if (mResumedActivity == next && next.isState(RESUMED)
            && taskDisplayArea.allResumedActivitiesComplete()) {
        ...
        return false;
    }

    if (!next.canResumeByCompat()) {
        return false;
    }

    // If we are currently pausing an activity, then don't do anything until that is done.
    //如果有正在Pause的Activity，在它Pause完成前不要做任何事
    final boolean allPausedComplete = mRootWindowContainer.allPausedActivitiesComplete();
    if (!allPausedComplete) {
        return false;
    }

    ...

    // Make sure that the user who owns this activity is started.  If not,
    // we will just leave it as is because someone should be bringing
    // another user's activities to the top of the stack.
    //确保拥有此Activity的用户已启动
    if (!mAtmService.mAmInternal.hasStartedUserState(next.mUserId)) {
        return false;
    }

    // The activity may be waiting for stop, but that is no longer
    // appropriate for it.
    mStackSupervisor.mStoppingActivities.remove(next);
    next.setSleeping(false);

    //这里似乎重复检查了，我去查看了一下master分支的代码，已经没有这一段了
    if (!mRootWindowContainer.allPausedActivitiesComplete()) {
        return false;
    }

    //设置启动Activity UID，获取WakeLock，保证在显示Activity的过程中，系统不会进行休眠状态
    mStackSupervisor.setLaunchSource(next.info.applicationInfo.uid);

    ActivityRecord lastResumed = null;
    //上一个具有焦点的ActivityStack（目前仍是屏幕上正在显示的那个Activity的栈）
    final ActivityStack lastFocusedStack = taskDisplayArea.getLastFocusedStack();
    if (lastFocusedStack != null && lastFocusedStack != this) {
        lastResumed = lastFocusedStack.mResumedActivity;
        ...
    }

    //Pause掉其他ActivityStack中的栈顶状态为Resume的Activity
    boolean pausing = taskDisplayArea.pauseBackStacks(userLeaving, next);
    //Pause掉当前ActivityStack中的栈顶状态为Resume的Activity
    if (mResumedActivity != null) {
        pausing |= startPausingLocked(userLeaving, false /* uiSleeping */, next);
    }
    if (pausing) { //有Activity执行了Pause
        // At this point we want to put the upcoming activity's process
        // at the top of the LRU list, since we know we will be needing it
        // very soon and it would be a waste to let it get killed if it
        // happens to be sitting towards the end.
        if (next.attachedToProcess()) {
            //将启动的Activity进程信息移至lru列表的头部
            //因为很快就会使用它启动Activity
            next.app.updateProcessInfo(false /* updateServiceConnectionActivities */,
                    true /* activityChange */, false /* updateOomAdj */,
                    false /* addPendingTopUid */);
        } else if (!next.isProcessRunning()) { //App进程未启动
            // Since the start-process is asynchronous, if we already know the process of next
            // activity isn't running, we can start the process earlier to save the time to wait
            // for the current activity to be paused.
            final boolean isTop = this == taskDisplayArea.getFocusedStack();
            //启动App进程
            mAtmService.startProcessAsync(next, false /* knownToBeDead */, isTop,
                    isTop ? "pre-top-activity" : "pre-activity");
        }
        ...
        //这里会先结束掉启动Activity的流程，等待onPause生命周期走完后
        //再重新调用这个方法执行下一步操作，避免Activity生命周期紊乱
        return true;
    } else if (mResumedActivity == next && next.isState(RESUMED)
            && taskDisplayArea.allResumedActivitiesComplete()) {
        // It is possible for the activity to be resumed when we paused back stacks above if the
        // next activity doesn't have to wait for pause to complete.
        // So, nothing else to-do except:
        // Make sure we have executed any pending transitions, since there
        // should be nothing left to do at this point.
        //不需要等待其他的Activity onPause完成
        //执行完切换效果后就没有什么其他需要做的了
        executeAppTransition(options);
        return true;
    }

    ...

    //启动Activity流程不会进入这个case
    if (prev != null && prev != next && next.nowVisible) {
        if (prev.finishing) {
            prev.setVisibility(false);
        }
    }

    //修改启动Activity的package的状态
    mAtmService.getPackageManager().setPackageStoppedState(
                next.packageName, false, next.mUserId);

    ... //Activity转场动画准备

    if (next.attachedToProcess()) { //对于将要启动的ActivityRecord来说，此时尚未完成和Process的绑定，返回false
        ...
        ActivityRecord lastResumedActivity =
                lastFocusedStack == null ? null : lastFocusedStack.mResumedActivity;
        //保存状态以做后面恢复使用
        final ActivityState lastState = next.getState();

        mAtmService.updateCpuStats();

        //更新ActivityRecord状态
        next.setState(RESUMED, "resumeTopActivityInnerLocked");

        //更新启动Activity的进程信息并将其移至lru列表的头部
        next.app.updateProcessInfo(false /* updateServiceConnectionActivities */,
                true /* activityChange */, true /* updateOomAdj */,
                true /* addPendingTopUid */);

        ... //更新Activity显示、方向和Configuration

        try {
            final ClientTransaction transaction =
                    ClientTransaction.obtain(next.app.getThread(), next.appToken);
            // Deliver all pending results.
            ArrayList<ResultInfo> a = next.results;
            if (a != null) {
                final int N = a.size();
                if (!next.finishing && N > 0) {
                    if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                            "Delivering results to " + next + ": " + a);
                    transaction.addCallback(ActivityResultItem.obtain(a));
                }
            }

            if (next.newIntents != null) {
                transaction.addCallback(
                        NewIntentItem.obtain(next.newIntents, true /* resume */));
            }

            // Well the app will no longer be stopped.
            // Clear app token stopped state in window manager if needed.
            next.notifyAppResumed(next.stopped);

            EventLogTags.writeWmResumeActivity(next.mUserId, System.identityHashCode(next),
                    next.getTask().mTaskId, next.shortComponentName);

            next.setSleeping(false);
            mAtmService.getAppWarningsLocked().onResumeActivity(next);
            next.app.setPendingUiCleanAndForceProcessStateUpTo(mAtmService.mTopProcessState);
            next.clearOptionsLocked();
            //设置onResume生命周期请求
            transaction.setLifecycleStateRequest(
                    ResumeActivityItem.obtain(next.app.getReportedProcState(),
                            dc.isNextTransitionForward()));
            //调度执行Activity onResume生命周期
            mAtmService.getLifecycleManager().scheduleTransaction(transaction);

            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Resumed "
                    + next);
        } catch (Exception e) { //resume失败需要尝试restart
            //恢复初始状态
            next.setState(lastState, "resumeTopActivityInnerLocked");
            if (lastResumedActivity != null) {
                lastResumedActivity.setState(RESUMED, "resumeTopActivityInnerLocked");
            }
            ...
            //重新启动Activity
            mStackSupervisor.startSpecificActivity(next, true, false);
            return true;
        }

        // From this point on, if something goes wrong there is no way
        // to recover the activity.
        try {
            //更新信息
            next.completeResumeLocked();
        } catch (Exception e) {
            // If any exception gets thrown, toss away this
            // activity and try the next one.
            Slog.w(TAG, "Exception thrown during resume of " + next, e);
            next.finishIfPossible("resume-exception", true /* oomAdj */);
            return true;
        }
    } else { //尚未绑定Process
        // Whoops, need to restart this activity!
        if (!next.hasBeenLaunched) {
            next.hasBeenLaunched = true;
        } else {
            if (SHOW_APP_STARTING_PREVIEW) {
                next.showStartingWindow(null /* prev */, false /* newTask */,
                        false /* taskSwich */);
            }
        }
        //启动Activity
        mStackSupervisor.startSpecificActivity(next, true, true);
    }

    return true;
}
```

这里的代码很长，其实我们只需要关注三个点：

1. `Pause`掉其他`Activity`

2. 如果对应`App`尚未启动，启动`App`进程

3. 启动`Activity`

第一步，在启动`Activity`前，我们需要先`Pause`掉其他`Activity`，这一点很好理解，我们可以对照着看`Activity`生命周期也是这样的，这里会通过`ATMS`跨进程调用相应`Activity`的`onPause`生命周期，等待`onPause`执行完成后，再跨进程调用回`ATMS`，经过一系列方法调用，又重新调用`resumeTopActivityInnerLocked`方法，继续执行下一步操作

第二步，如果`App`尚未启动，则先去启动`App`进程，这主要体现在这里

```java
if (!next.isProcessRunning()) { //App进程未启动
    //启动App进程
    mAtmService.startProcessAsync(next, false /* knownToBeDead */, isTop,
            isTop ? "pre-top-activity" : "pre-activity");
}
```

在启动完`App`进程后，会调用`ATMS`的`attachApplication`方法，最终调用到`ActivityStackSupervisor.realStartActivityLocked`方法启动`Activity`，这个方法后面会讲

第三步，如果`App`进程已经启动，这时候会调用`ActivityStackSupervisor.startSpecificActivity`方法，最终殊途同归调用`ActivityStackSupervisor.realStartActivityLocked`方法启动`Activity`

# 结束

这里篇幅有点过长了，所以我觉得还是分篇比较好，这一章其实重要内容不是很多，最主要的内容，像`Activity`的生命周期控制，`App`进程的启动，`Activity`具体的启动及其后续生命周期执行都会放在下一章来讲