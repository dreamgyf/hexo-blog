---
title: Android开发常见问题总结（持续更新）
date: 2021-04-19 16:47:00
tags: 常见问题
categories: 
- [Android, 常见问题]
---

# 滑动

1. 滑动嵌套

滑动组件的嵌套可能会产生以下一些问题：

- 滑动冲突

解决方法：使用`NestedScrollView`替代`ScrollView`，`RecyclerView`可以设置属性`android:nestedScrollingEnabled="false"`或代码里`setNestedScrollingEnabled(false);`来禁用组件自身的滑动

注意：如果`RecyclerView`只能显示一个Item的话，需要设置`NestedScrollView`的属性`android:fillViewport="true"`

- 滑动失效

`ScrollView`设置`fillViewport="true"`的情况下，如果对`ScrollView`的直接子view设置上下margin，在超出内容的高度小于设置的margin的情况下，可能会导致整个`ScrollView`滑动失效

2. 焦点抢占

`ScrollView`、`RecyclerView`等滑动组件可能会抢占焦点，导致界面显示时直接滑动到对应组件的位置，而不是顶部

解决方法：在顶部View(或者其他你所期望的初始位置)加上属性`android:focusable="true"`和`android:focusableInTouchMode="true"`

新解决方法：在顶部View上加`android:descendantFocusability`属性，该属性是用来定义父布局与子布局之间的关系的，它有三种值：

- `beforeDescendants`：父布局会优先其子类控件而获取到焦点
- `afterDescendants`：父布局只有当其子类控件不需要获取焦点时才获取焦点
- `blocksDescendants`：父布局会覆盖子类控件而直接获得焦点

使用`blocksDescendants`覆盖子布局焦点以解决焦点抢占问题

---

# RecyclerView

## Adapter

1. 在`onBindViewHolder`中设置子View回调时需要注意

如果回调的参数包括position时，需要注意有没有地方会调用`notifyItemRemoved`或`notifyItemRangeRemoved`，如果有，需要使用`holder.getAdapterPosition()`来代替`onBindViewHolder`方法的position参数

原因：`notifyItemRemoved`不会对其他的Item重新调用`onBindViewHolder`，这样可能会导致position错位。`holder.getAdapterPosition()`方法会返回数据在 Adapter 中的位置（即使位置的变化还未刷新到布局中）

2. 如何在更新数据后重新定位到顶部

```java
//重写父类方法，获得绑定的RecyclerView
@Override
public void onAttachedToRecyclerView(@NonNull RecyclerView recyclerView) {
	super.onAttachedToRecyclerView(recyclerView);
	mRecyclerView = recyclerView;
}

//当数据更新后调用
if (mRecyclerView != null && mRecyclerView.getChildCount() > 0) {
	mRecyclerView.scrollToPosition(0);
}
```

之前尝试过`mRecyclerView.scrollTo(0, 0);`但没有起效，不清楚为什么

3. 动态部分更新数据时

如果`RecyclerView`需要动态更新**部分**数据，并且在`onBindViewHolder`时对某些view设置了事件或者回调等，如果此时使用到了position参数需要注意，如果你只notify了部分数据更新，可能会导致更新后部分ViewHolder中的回调里的position不正确，建议：

- 使用`notifyDataSetChanged()`
- 使用`notifyItem`，但是在`onBindViewHolder`中设置回调时不要使用position参数，而是使用`holder.getAdapterPosition()`替代（注意这个方法在`ViewHolder`没有和`RecyclerView`绑定时会返回-1 `NO_POSITION`）

## ItemDecoration

1. `StaggeredGridLayoutManager`下`ItemDecoration`的`offset`计算错误

主要是因为`RecyclerView`动态更新数据时，会执行多次`measure`，但只会在第一次`measure`的时候调用`ItemDecoration.getItemOffsets`（因为`LP`里的`mInsetsDirty`变量），此时获得的`spanIndex`是一个错误值

这个问题的具体分析可以看[这篇文章](https://blog.kyleduo.com/2017/07/27/recyclerview-wrong-decoration-inset/)，暂时没有什么好的解决方案，不建议大家使用反射，毕竟你不知道`Android`会不会更改这个变量

---

# Dialog

1. 生命周期

- 初始化时需要注意

Dialog在第一次调用`show()`方法后才会执行`onCreate(Bundle savedInstanceState)`方法，因此建议自定义Dialog时将`findViewById`等初始化操作放在构造函数中进行，避免外部使用时因在`show()`之前设置视图数据导致NPE

---

# PopupWindow

1. 点击没反应

`PopupWindow`如果不设置背景的话，在某些5.x以下系统机型上会出现点击没反应的问题

解决方法：给PopupWindow设置一个空背景`popupWindow.setBackgroundDrawable(new BitmapDrawable(mContext.getResources(), (Bitmap) null));`

详见：https://juejin.cn/post/6844903761488379912

---

# 广播

1. 隐式广播

在Android8.0以上的系统，大部分的隐式广播都被限制不可使用。

解决方法：
1. 使用动态广播
2. 使用显示广播
```java
// 方式一: 设置Component
Intent intent = new Intent(SOME_ACTION);
intent.setComponent(new ComponentName(context, SomeReceiver.class));
context.sendBroadcast(intent);

// 方式二: 设置Package
Intent intent = new Intent(SOME_ACTION);
intent.setPackage("com.dreamgyf.xxx");
context.sendBroadcast(intent);

// 不知道包名的话可以通过PackageManager获取所有注册了指定action的广播的package
Intent actionIntent = new Intent(SOME_ACTION);
PackageManager pm = context.getPackageManager();
List<ResolveInfo> matches = pm.queryBroadcastReceivers(actionIntent, 0);
for (ResolveInfo resolveInfo : matches) {            
    Intent intent = new Intent(actionIntent);            
    intent.setPackage(resolveInfo.activityInfo.applicationInfo.packageName);            
    intent.setAction(SOME_ACTION);            
    context.sendBroadcast(intent);        
}
```
---

# 实体键盘

1. `EditText`有焦点时会拦截键盘的数字键

解决方法：使用`TextWatcher`等监听`EditText`输入

---

# 内存泄漏

1. 动画

在Activity销毁之前如果没有cancel掉，会导致这个Activity内存泄漏

2. `ClickableSpan`

使用`SpannableString.setSpan`方法设置`ClickableSpan`可能导致内存泄漏

原因：`TextView`在`onSaveInstanceState`时会将`ClickableSpan`复制一份，由于某些原因，`SpannableString`不会删除这个`ClickableSpan`，从而导致内存泄漏，详见：
[StackOverflow](https://stackoverflow.com/questions/28539216/android-textview-leaks-with-setmovementmethod)

解决方法：自定义一个抽象类同时继承`ClickableSpan`和实现`NoCopySpan`接口，外部`setSpan`时使用这个抽象类

---

# Fragment

1. `Fragment`尽量不要使用带参构造函数，一定要保证有一个不含参的构造函数，否则在`Activity`重建时尝试反射`newInstance`恢复`Fragment`时会抛出`Could not find Fragment constructor`异常

---

# 混淆

1. 反射

如果使用到了反射，需要特别注意需不需要在`proguard-rules`中加入keep规则

2. module混淆

如果是多module项目，想要在module中增加混淆规则，`proguardFiles`属性是无效的，应该使用`consumerProguardFiles`属性

```groovy
android {
    compileSdkVersion 28

    defaultConfig {
        minSdkVersion 21
        targetSdkVersion 28
        versionName repo.version

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"

        consumerProguardFiles 'proguard-rules.pro' //这里
    }
    ...
}
```

---

# 相机开发

1. 拍照角度

相机的方向一般是以手机横向作为正方向，这样如果我们以竖屏的方式拍照，拍出来的照片可能会出现旋转了90度的情况，这时候就需要在拍照完后处理一下图片，旋转到正确位置。

具体介绍与算法在Android SDK中`CaptureRequest.JPEG_ORIENTATION`的注释中

```java
private int getJpegOrientation(CameraCharacteristics c, int deviceOrientation) {
    if (deviceOrientation == android.view.OrientationEventListener.ORIENTATION_UNKNOWN)
        return 0;
    //获得相机方向与设备方向间的夹角
    int sensorOrientation = c.get(CameraCharacteristics.SENSOR_ORIENTATION);

    // Round device orientation to a multiple of 90
    deviceOrientation = (deviceOrientation + 45) / 90 * 90;

    // Reverse device orientation for front-facing cameras
    boolean facingFront = c.get(CameraCharacteristics.LENS_FACING) == CameraCharacteristics.LENS_FACING_FRONT;
    if (facingFront) deviceOrientation = -deviceOrientation;

    // Calculate desired JPEG orientation relative to camera orientation to make
    // the image upright relative to the device orientation
    int jpegOrientation = (sensorOrientation + deviceOrientation + 360) % 360;

    return jpegOrientation;
}
```

计算好角度后就可以对图片做旋转了，网上有很多文章都说使用这种方式做旋转

```java
captureBuilder.set(CaptureRequest.JPEG_ORIENTATION, getJpegOrientation(deviceRotation));
```

但实际上在某些系统上 (MIUI)，设置的这个参数并不会生效，所以我的方案是，获得拍摄好的照片Bitmap后，再对其进行旋转

```java
public Bitmap rotateBitmap(Bitmap bitmap, int angle) {
   Matrix matrix = new Matrix();
   matrix.setRotate(angle);
   return Bitmap.createBitmap(bitmap, 0, 0, bitmap.getWidth(), bitmap.getHeight(), matrix, true);
}
```