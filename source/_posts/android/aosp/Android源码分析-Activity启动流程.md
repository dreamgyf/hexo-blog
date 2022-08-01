---
title: Android源码分析 - Activity启动流程
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

```java
int execute() {
    try {
        // Refuse possible leaked file descriptors
        if (mRequest.intent != null && mRequest.intent.hasFileDescriptors()) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        final LaunchingState launchingState;
        synchronized (mService.mGlobalLock) {
            final ActivityRecord caller = ActivityRecord.forTokenLocked(mRequest.resultTo);
            launchingState = mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(
                    mRequest.intent, caller);
        }

        // If the caller hasn't already resolved the activity, we're willing
        // to do so here. If the caller is already holding the WM lock here,
        // and we need to check dynamic Uri permissions, then we're forced
        // to assume those permissions are denied to avoid deadlocking.
        if (mRequest.activityInfo == null) {
            mRequest.resolveActivity(mSupervisor);
        }

        int res;
        synchronized (mService.mGlobalLock) {
            final boolean globalConfigWillChange = mRequest.globalConfig != null
                    && mService.getGlobalConfiguration().diff(mRequest.globalConfig) != 0;
            final ActivityStack stack = mRootWindowContainer.getTopDisplayFocusedStack();
            if (stack != null) {
                stack.mConfigWillChange = globalConfigWillChange;
            }
            if (DEBUG_CONFIGURATION) {
                Slog.v(TAG_CONFIGURATION, "Starting activity when config will change = "
                        + globalConfigWillChange);
            }

            final long origId = Binder.clearCallingIdentity();

            res = resolveToHeavyWeightSwitcherIfNeeded();
            if (res != START_SUCCESS) {
                return res;
            }
            res = executeRequest(mRequest);

            Binder.restoreCallingIdentity(origId);

            if (globalConfigWillChange) {
                // If the caller also wants to switch to a new configuration, do so now.
                // This allows a clean switch, as we are waiting for the current activity
                // to pause (so we will not destroy it), and have not yet started the
                // next activity.
                mService.mAmInternal.enforceCallingPermission(
                        android.Manifest.permission.CHANGE_CONFIGURATION,
                        "updateConfiguration()");
                if (stack != null) {
                    stack.mConfigWillChange = false;
                }
                if (DEBUG_CONFIGURATION) {
                    Slog.v(TAG_CONFIGURATION,
                            "Updating to new configuration after starting activity.");
                }
                mService.updateConfigurationLocked(mRequest.globalConfig, null, false);
            }

            // Notify ActivityMetricsLogger that the activity has launched.
            // ActivityMetricsLogger will then wait for the windows to be drawn and populate
            // WaitResult.
            mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(launchingState, res,
                    mLastStartActivityRecord);
            return getExternalResult(mRequest.waitResult == null ? res
                    : waitForResult(res, mLastStartActivityRecord));
        }
    } finally {
        onExecutionComplete();
    }
}
```