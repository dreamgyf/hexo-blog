---
title: Android动态权限申请从未如此简单
date: 2023-04-23 21:55:46
tags: 
- Android
- 权限
categories: 
- [Android, 权限]
---

# 前言

**注：只想看实现的朋友们可以直接跳到最后面的最终实现**

大家是否还在为动态权限申请感到苦恼呢？传统的动态权限申请需要在`Activity`中重写`onRequestPermissionsResult`方法来接收用户权限授予的结果。试想一下，你需要在一个子模块中申请权限，那得从这个模块所在的`Activity`的`onRequestPermissionsResult`中将结果一层层再传回到这个模块中，相当的麻烦，代码也相当冗余和不干净，逼死强迫症。

# 使用

为了解决这个痛点，我封装出了两个方法，用于随时随地快速的动态申请权限，我们先来看看我们的封装方法是如何调用的：

```kotlin
activity.requestPermission(Manifest.permission.CAMERA, onPermit = {
    //申请权限成功 Do something
}, onDeny = { shouldShowCustomRequest ->
    //申请权限失败 Do something
    if (shouldShowCustomRequest) {
        //用户选择了拒绝并且不在询问，此时应该使用自定义弹窗提醒用户授权（可选）
    }
})
```

这样是不是非常的简单便捷？申请和结果回调都在一个方法内处理，并且支持随用随调。

# 方案

那么，这么方便好用的方法是怎么实现的呢？不知道小伙伴们在平时开发中有没有注意到过，当你调用`startActivityForResult`时，AS会提示你该方法已被弃用，点进去看会告诉你应该使用`registerForActivityResult`方法替代。没错，这就是`androidx`给我们提供的`ActivityResult`功能，并且这个功能不仅支持`ActivityResult`回调，还支持打开文档，拍摄照片，选择文件等各种各样的回调，同样也包括我们今天要说的权限申请

其实Android在官方文档 [请求运行时权限](https://developer.android.com/training/permissions/requesting?hl=zh-cn) 中就已经将其作为动态权限申请的推荐方法了，如下示例代码所示：

```kotlin
val requestPermissionLauncher =
    registerForActivityResult(RequestPermission()
    ) { isGranted: Boolean ->
        if (isGranted) {
            // Permission is granted. Continue the action or workflow in your
            // app.
        } else {
            // Explain to the user that the feature is unavailable because the
            // feature requires a permission that the user has denied. At the
            // same time, respect the user's decision. Don't link to system
            // settings in an effort to convince the user to change their
            // decision.
        }
    }

when {
    ContextCompat.checkSelfPermission(
            CONTEXT,
            Manifest.permission.REQUESTED_PERMISSION
            ) == PackageManager.PERMISSION_GRANTED -> {
        // You can use the API that requires the permission.
    }
    shouldShowRequestPermissionRationale(...) -> {
        // In an educational UI, explain to the user why your app requires this
        // permission for a specific feature to behave as expected, and what
        // features are disabled if it's declined. In this UI, include a
        // "cancel" or "no thanks" button that lets the user continue
        // using your app without granting the permission.
        showInContextUI(...)
    }
    else -> {
        // You can directly ask for the permission.
        // The registered ActivityResultCallback gets the result of this request.
        requestPermissionLauncher.launch(
                Manifest.permission.REQUESTED_PERMISSION)
    }
}
```

说到这里，可能有小伙伴要质疑我了：“官方文档里都写明了的东西，你还特地写一遍，还起了这么个标题，是不是在水文章？！”

莫急，如果你遵照以上方法这么写的话，在实际调用的时候会直接发生崩溃：

```txt
java.lang.IllegalStateException: 
LifecycleOwner Activity is attempting to register while current state is RESUMED. 
LifecycleOwners must call register before they are STARTED.
```

这段报错很明显的告诉我们，我们的注册工作必须要在`Activity`声明周期`STARTED`之前进行（也就是`onCreate`时和`onStart`完成前），但这样我们就必须要事先注册好所有可能会用到的权限，没办法做到随时随地有需要时再申请权限了，有办法解决这个问题吗？答案是肯定的。

# 绕过生命周期检测

想解决这个问题，我们必须要知道问题的成因，让我们带着问题进到源码中一探究竟：

```java
public final <I, O> ActivityResultLauncher<I> registerForActivityResult(
        @NonNull ActivityResultContract<I, O> contract,
        @NonNull ActivityResultCallback<O> callback) {
    return registerForActivityResult(contract, mActivityResultRegistry, callback);
}

public final <I, O> ActivityResultLauncher<I> registerForActivityResult(
        @NonNull final ActivityResultContract<I, O> contract,
        @NonNull final ActivityResultRegistry registry,
        @NonNull final ActivityResultCallback<O> callback) {
    return registry.register(
            "activity_rq#" + mNextLocalRequestCode.getAndIncrement(), this, contract, callback);
}

public final <I, O> ActivityResultLauncher<I> register(
        @NonNull final String key,
        @NonNull final LifecycleOwner lifecycleOwner,
        @NonNull final ActivityResultContract<I, O> contract,
        @NonNull final ActivityResultCallback<O> callback) {

    Lifecycle lifecycle = lifecycleOwner.getLifecycle();

    if (lifecycle.getCurrentState().isAtLeast(Lifecycle.State.STARTED)) {
        throw new IllegalStateException("LifecycleOwner " + lifecycleOwner + " is "
                + "attempting to register while current state is "
                + lifecycle.getCurrentState() + ". LifecycleOwners must call register before "
                + "they are STARTED.");
    }

    registerKey(key);
    LifecycleContainer lifecycleContainer = mKeyToLifecycleContainers.get(key);
    if (lifecycleContainer == null) {
        lifecycleContainer = new LifecycleContainer(lifecycle);
    }
    LifecycleEventObserver observer = new LifecycleEventObserver() { ... };
    lifecycleContainer.addObserver(observer);
    mKeyToLifecycleContainers.put(key, lifecycleContainer);

    return new ActivityResultLauncher<I>() { ... };
}
```

我们可以发现，`registerForActivityResult`实际上就是调用了`ComponentActivity`内部成员变量的`mActivityResultRegistry.register`方法，而在这个方法的一开头就检查了当前`Activity`的生命周期，如果生命周期位于`STARTED`后则直接抛出异常，那我们该如何绕过这个限制呢？

其实在`register`方法的下面就有一个同名重载方法，这个方法并没有做生命周期的检测：

```java
public final <I, O> ActivityResultLauncher<I> register(
        @NonNull final String key,
        @NonNull final ActivityResultContract<I, O> contract,
        @NonNull final ActivityResultCallback<O> callback) {
    registerKey(key);
    mKeyToCallback.put(key, new CallbackAndContract<>(callback, contract));

    if (mParsedPendingResults.containsKey(key)) {
        @SuppressWarnings("unchecked")
        final O parsedPendingResult = (O) mParsedPendingResults.get(key);
        mParsedPendingResults.remove(key);
        callback.onActivityResult(parsedPendingResult);
    }
    final ActivityResult pendingResult = mPendingResults.getParcelable(key);
    if (pendingResult != null) {
        mPendingResults.remove(key);
        callback.onActivityResult(contract.parseResult(
                pendingResult.getResultCode(),
                pendingResult.getData()));
    }

    return new ActivityResultLauncher<I>() { ... };
}
```

找到这个方法就简单了，我们将`registerForActivityResult`方法调用替换成`activityResultRegistry.register`调用就可以了

当然，我们还需要注意一些小细节，检查生命周期的`register`方法同时也会注册生命周期回调，当`Activity`被销毁时会将我们注册的`ActivityResult`回调移除，我们也需要给我们封装的方法加上这个逻辑，最终实现就如下所示。

# 最终实现

```kotlin
private val nextLocalRequestCode = AtomicInteger()

private val nextKey: String
    get() = "activity_rq#${nextLocalRequestCode.getAndIncrement()}"

fun ComponentActivity.requestPermission(
    permission: String,
    onPermit: () -> Unit,
    onDeny: (shouldShowCustomRequest: Boolean) -> Unit
) {
    if (ContextCompat.checkSelfPermission(this, permission) == PackageManager.PERMISSION_GRANTED) {
        onPermit()
        return
    }
    var launcher by Delegates.notNull<ActivityResultLauncher<String>>()
    launcher = activityResultRegistry.register(
        nextKey,
        ActivityResultContracts.RequestPermission()
    ) { result ->
        if (result) {
            onPermit()
        } else {
            onDeny(!ActivityCompat.shouldShowRequestPermissionRationale(this, permission))
        }
        launcher.unregister()
    }
    lifecycle.addObserver(object : LifecycleEventObserver {
        override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
            if (event == Lifecycle.Event.ON_DESTROY) {
                launcher.unregister()
                lifecycle.removeObserver(this)
            }
        }
    })
    launcher.launch(permission)
}

fun ComponentActivity.requestPermissions(
    permissions: Array<String>,
    onPermit: () -> Unit,
    onDeny: (shouldShowCustomRequest: Boolean) -> Unit
) {
    var hasPermissions = true
    for (permission in permissions) {
        if (ContextCompat.checkSelfPermission(
                this,
                permission
            ) != PackageManager.PERMISSION_GRANTED
        ) {
            hasPermissions = false
            break
        }
    }
    if (hasPermissions) {
        onPermit()
        return
    }
    var launcher by Delegates.notNull<ActivityResultLauncher<Array<String>>>()
    launcher = activityResultRegistry.register(
        nextKey,
        ActivityResultContracts.RequestMultiplePermissions()
    ) { result ->
        var allAllow = true
        for (allow in result.values) {
            if (!allow) {
                allAllow = false
                break
            }
        }
        if (allAllow) {
            onPermit()
        } else {
            var shouldShowCustomRequest = false
            for (permission in permissions) {
                if (!ActivityCompat.shouldShowRequestPermissionRationale(this, permission)) {
                    shouldShowCustomRequest = true
                    break
                }
            }
            onDeny(shouldShowCustomRequest)
        }
        launcher.unregister()
    }
    lifecycle.addObserver(object : LifecycleEventObserver {
        override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
            if (event == Lifecycle.Event.ON_DESTROY) {
                launcher.unregister()
                lifecycle.removeObserver(this)
            }
        }
    })
    launcher.launch(permissions)
}
```

# 总结

其实很多实用技巧本质上都是很简单的，但没有接触过就很难想到，我将我的开发经验分享给大家，希望能帮助到大家。