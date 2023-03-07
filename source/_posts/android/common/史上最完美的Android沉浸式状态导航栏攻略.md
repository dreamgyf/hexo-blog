---
title: 史上最完美的Android沉浸式状态导航栏攻略
date: 2023-01-07 20:51:52
tags: 
- Android
- 沉浸式
- 状态栏
- 导航栏
- StatusBar
- NavigationBar
categories: 
- [Android, 沉浸式]
- [Android, 状态栏]
- [Android, 导航栏]
- [Android, StatusBar]
- [Android, NavigationBar]
---

# 前言

最近我在小破站开发一款新App，叫**高能链**。我是一个完美主义者，所以不管对架构还是UI，我都是比较抠细节的，在状态栏和导航栏沉浸式这一块，我还是踩了挺多坑，费了挺多精力的。这次我将我踩坑，适配各机型总结出来的史上最完美的Android沉浸式状态导航栏攻略分享给大家，大家也可以去 **[高能链官网](https://www.upowerchain.com/)** 下载体验一下我们的App，实际感受一下沉浸式状态导航栏的效果（登录，实名等账号相关页面由于不是我开发的，就没有适配沉浸式导航栏啦，嘻嘻）

**注：此攻略只针对 Android 5.0 及以上机型，即 minSdkVersion >= 21**

# 实际效果

在开始攻略之前，我们先看看完美的沉浸式状态导航栏效果

## 传统三键式导航栏

![客态列表页_三键式](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/%E5%8F%B2%E4%B8%8A%E6%9C%80%E5%AE%8C%E7%BE%8E%E7%9A%84Android%E6%B2%89%E6%B5%B8%E5%BC%8F%E7%8A%B6%E6%80%81%E5%AF%BC%E8%88%AA%E6%A0%8F%E6%94%BB%E7%95%A5_%E5%AE%A2%E6%80%81%E5%88%97%E8%A1%A8%E9%A1%B5_%E4%B8%89%E9%94%AE%E5%BC%8F.jpg)

![客态详情页_三键式](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/%E5%8F%B2%E4%B8%8A%E6%9C%80%E5%AE%8C%E7%BE%8E%E7%9A%84Android%E6%B2%89%E6%B5%B8%E5%BC%8F%E7%8A%B6%E6%80%81%E5%AF%BC%E8%88%AA%E6%A0%8F%E6%94%BB%E7%95%A5_%E5%AE%A2%E6%80%81%E8%AF%A6%E6%83%85%E9%A1%B5_%E4%B8%89%E9%94%AE%E5%BC%8F.jpg)

## 全面屏导航条

![客态列表页_导航条](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/%E5%8F%B2%E4%B8%8A%E6%9C%80%E5%AE%8C%E7%BE%8E%E7%9A%84Android%E6%B2%89%E6%B5%B8%E5%BC%8F%E7%8A%B6%E6%80%81%E5%AF%BC%E8%88%AA%E6%A0%8F%E6%94%BB%E7%95%A5_%E5%AE%A2%E6%80%81%E5%88%97%E8%A1%A8%E9%A1%B5_%E5%AF%BC%E8%88%AA%E6%9D%A1.jpg)

![客态详情页_导航条](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/%E5%8F%B2%E4%B8%8A%E6%9C%80%E5%AE%8C%E7%BE%8E%E7%9A%84Android%E6%B2%89%E6%B5%B8%E5%BC%8F%E7%8A%B6%E6%80%81%E5%AF%BC%E8%88%AA%E6%A0%8F%E6%94%BB%E7%95%A5_%E5%AE%A2%E6%80%81%E8%AF%A6%E6%83%85%E9%A1%B5_%E5%AF%BC%E8%88%AA%E6%9D%A1.jpg)

# 理论分析

在上具体实现代码之前，我们先分析一下，实现沉浸式状态导航栏需要几步

1. 状态栏导航栏底色透明

2. 根据当前页面的背景色，给状态栏字体和导航栏按钮（或导航条）设置亮色或暗色

3. 状态栏导航栏设置透明后，我们页面的布局会延伸到原本状态栏导航栏的位置，这时候需要一些手段将我们需要显示的正文内容回缩到其正确的显示范围内

    这里我给大家提供以下几种思路，大家可以根据实际情况自行选择：

   - 设置`fitsSystemWindows`属性
   - 根据状态栏导航栏的高度，给根布局设置相应的`paddingTop`和`paddingBottom`
   - 根据状态栏导航栏的高度，给需要移位的控件设置相应的`marginTop`和`marginBottom`
   - 在顶部和底部增加两个占位的`View`，高度分别设置成状态栏和导航栏的高度
   - 针对滑动视图，巧用`clipChildren`和`clipToPadding`属性（可参照高能链藏品详情页样式）

# 沉浸式状态栏

思路说完了，我们现在开始进入实战，沉浸式状态栏比较简单，没什么坑

## 状态栏透明

首先第一步，我们需要将状态栏的背景设置为透明，这里我直接放代码

```kotlin
fun transparentStatusBar(window: Window) {
    window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)
    window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS)
    var systemUiVisibility = window.decorView.systemUiVisibility
    systemUiVisibility =
        systemUiVisibility or View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN or View.SYSTEM_UI_FLAG_LAYOUT_STABLE
    window.decorView.systemUiVisibility = systemUiVisibility
    window.statusBarColor = Color.TRANSPARENT

    //设置状态栏文字颜色
    setStatusBarTextColor(window, NightMode.isNightMode(window.context))
}
```

首先，我们需要将`FLAG_TRANSLUCENT_STATUS`这个`windowFlag`换成`FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS`，否则状态栏不会完全透明，会有一个半透明的灰色蒙层

`FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS`这个`flag`表示系统`Bar`的背景将交给当前`window`绘制

`SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN`这个`flag`表示`Activity`全屏显示，但状态栏不会被隐藏，依然可见

`SYSTEM_UI_FLAG_LAYOUT_STABLE`这个`flag`表示保持整个`View`稳定，使`View`不会因为系统UI的变化而重新`layout`

`SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN`和`SYSTEM_UI_FLAG_LAYOUT_STABLE`这两个`flag`通常是一起使用的，我们设置这两个`flag`，然后再将`statusBarColor`设置为透明，就达成了状态栏背景透明的效果

## 状态栏文字颜色

接着我们就该设置状态栏文字颜色了，细心的小伙伴们应该已经注意到了，我在`transparentStatusBar`方法的末尾加了一个`setStatusBarTextColor`的方法调用，一般情况下，如果是日间模式，页面背景通常都是亮色，所以此时状态栏文字颜色设置为黑色比较合理，而在夜间模式下，页面背景通常都是暗色，此时状态栏文字颜色设置为白色比较合理，对应代码如下

```kotlin
fun setStatusBarTextColor(window: Window, light: Boolean) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        var systemUiVisibility = window.decorView.systemUiVisibility
        systemUiVisibility = if (light) { //白色文字
            systemUiVisibility and View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR.inv()
        } else { //黑色文字
            systemUiVisibility or View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR
        }
        window.decorView.systemUiVisibility = systemUiVisibility
    }
}
```

`Android 8.0`以上才支持导航栏文字颜色的修改，`SYSTEM_UI_FLAG_LIGHT_STATUS_BAR`这个`flag`表示亮色状态栏，即黑色状态栏文字，所以如果希望状态栏文字为黑色，就设置这个`flag`，如果希望状态栏文字为白色，就将这个`flag`从`systemUiVisibility`中剔除

可能有小伙伴不太了解`kotlin`中的位运算，`kotlin`中的`or`、`and`、`inv`分别对应着或、与、取反运算

所以

```kotlin
systemUiVisibility and View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR.inv()
```

翻译成`java`即为

```java
systemUiVisibility & ~View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR
```

在原生系统上，这么设置就可以成功设置状态栏文字颜色，但我发现，在某些系统上，这样设置后的效果是不可预期的，譬如`MIUI`系统的状态栏文字颜色似乎是根据状态栏背景颜色自适应的，且日间模式和黑夜模式下的自适应策略还略有不同。不过在大多数情况下，它自适应的颜色都是正常的，我们就按照我们希望的结果设置就可以了。

## 矫正显示区域

### fitsSystemWindows

矫正状态栏显示区域最简单的办法就是设置`fitsSystemWindows`属性，设置了该属性的`View`的所有`padding`属性都将失效，并且系统会自动为其添加`paddingTop`（设置了透明状态栏的情况下）和`paddingBottom`（设置了透明导航栏的情况下）

我个人是不用这种方式的，首先它会覆盖你设置的`padding`，其次，如果你同时设置了透明状态栏和透明导航栏，这个属性没有办法分开来处理，很不灵活

### 获取状态栏高度

除了`fitsSystemWindows`这种方法外，其他的方法都得依靠获取状态栏高度了，这里直接上代码

```kotlin
fun getStatusBarHeight(context: Context): Int {
    val resId = context.resources.getIdentifier(
        "status_bar_height", "dimen", "android"
    )
    return context.resources.getDimensionPixelSize(resId)
}
```

状态栏不像导航栏那样多变，所以直接这样获取高度就可以了，导航栏的高度飘忽不定才是真正的噩梦

这里再给两个设置`View` `margin`或`padding`的工具方法吧，帮助大家快速使用

```kotlin
fun fixStatusBarMargin(vararg views: View) {
    views.forEach { view ->
        (view.layoutParams as? ViewGroup.MarginLayoutParams)?.let { lp ->
            lp.topMargin = lp.topMargin + getStatusBarHeight(view.context)
            view.requestLayout()
        }
    }
}

fun paddingByStatusBar(view: View) {
    view.setPadding(
        view.paddingLeft,
        view.paddingTop + getStatusBarHeight(view.context),
        view.paddingRight,
        view.paddingBottom
    )
}
```

# 沉浸式导航栏

沉浸式导航栏相比沉浸式状态栏坑会多很多，具体原因我们后面再说

## 导航栏透明

和沉浸式状态栏一样，第一步我们需要将导航栏的背景设置为透明

```kotlin
fun transparentNavigationBar(window: Window) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        window.isNavigationBarContrastEnforced = false
    }
    window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION)
    window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS)
    var systemUiVisibility = window.decorView.systemUiVisibility
    systemUiVisibility =
        systemUiVisibility or View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
    window.decorView.systemUiVisibility = systemUiVisibility
    window.navigationBarColor = Color.TRANSPARENT

    //设置导航栏按钮或导航条颜色
    setNavigationBarBtnColor(window, NightMode.isNightMode(window.context))
}
```

在`Android 10`以上，当设置了导航栏栏背景为透明时，`isNavigationBarContrastEnforced`如果为`true`，则系统会自动绘制一个半透明背景来提供对比度，所以我们要将这个属性设为`false`

ps：状态栏其实也有对应的属性`isStatusBarContrastEnforced`，只不过这个属性默认即为`false`，我们不需要特意去设置

## 导航栏按钮或导航条颜色

和设置状态栏文字颜色一样，我这里就不多介绍了

```kotlin
fun setNavigationBarBtnColor(window: Window, light: Boolean) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        var systemUiVisibility = window.decorView.systemUiVisibility
        systemUiVisibility = if (light) { //白色按钮
            systemUiVisibility and View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR.inv()
        } else { //黑色按钮
            systemUiVisibility or View.SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR
        }
        window.decorView.systemUiVisibility = systemUiVisibility
    }
}
```

## 矫正显示区域

### fitsSystemWindows

和状态栏使用一样，我就不重复说明了

### 获取导航栏高度

自从全面屏手势开始流行，导航栏也从原先的三键式，变成了三键式、导航条、全隐藏这三种情况，这三种情况下的高度也是互不相同的

三键式和导航条这两种情况我们都可以通过`android.R.dimen.navigation_bar_height`这个资源获取到准确高度，但现在很多系统都支持隐藏导航栏的功能，在这种情况下，虽然实际导航栏的高度应该是0，但是通过资源获取到的高度却为三键式或导航条的高度，这就给我们沉浸式导航栏的适配带来了很大困难

经过我的各种尝试，我发现只有一种方式可以准确的获取到当前导航栏的高度，那就是`WindowInsets`，至于`WindowInsets`是什么我就不多介绍了，我们直接看代码

```kotlin
/**
* 仅当view attach window后生效
*/
private fun getRealNavigationBarHeight(view: View): Int {
    val insets = ViewCompat.getRootWindowInsets(view)
        ?.getInsets(WindowInsetsCompat.Type.navigationBars())
    //WindowInsets为null则默认通过资源获取高度
    return insets?.bottom ?: getNavigationBarHeight(view.context)
}
```

这里需要注意到我在方法上写的注释，只有当`View`和`Window` attach 后，才能获得到`WindowInsets`，否则为`null`，所以我一开始的想法是先检查`View`是否 attach 了`Window`，如果有的话则直接调用`getRealNavigationBarHeight`方法，如果没有的话，调用`View.addOnAttachStateChangeListener`方法，当出发`attach`回调后，再调用`getRealNavigationBarHeight`方法获取高度

这种方式在大部分情况下运行良好，但在我一次无意中切换了系统夜间模式后发现，获取到的导航栏高度变成了0，并且这还是一个偶现的问题，于是我尝试使用`View.setOnApplyWindowInsetsListener`，监听`WindowInsets`的变化发现，这个回调有可能会触发多次，在触发多次的情况下，前几次的值都为0，只有最后一次的值为真正的导航栏高度

于是我准备用`View.setOnApplyWindowInsetsListener`代替`View.addOnAttachStateChangeListener`，但毕竟一个是setListener，一个是addListener，setListener有可能会把之前设置好的Listener覆盖，或者被别的Listener覆盖掉，再考虑到之后会提到的底部`Dialog`沉浸式导航栏适配的问题，我折中了一下，决定只对`Activity`下的`rootView`设置回调

以下是完整代码

```kotlin
private class NavigationViewInfo(
    val hostRef: WeakReference<View>,
    val viewRef: WeakReference<View>,
    val rawBottom: Int,
    val onNavHeightChangeListener: (View, Int, Int) -> Unit
)

private val navigationViewInfoList = mutableListOf<NavigationViewInfo>()

private val onApplyWindowInsetsListener = View.OnApplyWindowInsetsListener { v, insets ->
    val windowInsetsCompat = WindowInsetsCompat.toWindowInsetsCompat(insets, v)
    val navHeight =
        windowInsetsCompat.getInsets(WindowInsetsCompat.Type.navigationBars()).bottom
    val it = navigationViewInfoList.iterator()
    while (it.hasNext()) {
        val info = it.next()
        val host = info.hostRef.get()
        val view = info.viewRef.get()
        if (host == null || view == null) {
            it.remove()
            continue
        }

        if (host == v) {
            info.onNavHeightChangeListener(view, info.rawBottom, navHeight)
        }
    }
    insets
}

private val actionMarginNavigation: (View, Int, Int) -> Unit =
    { view, rawBottomMargin, navHeight ->
        (view.layoutParams as? ViewGroup.MarginLayoutParams)?.let {
            it.bottomMargin = rawBottomMargin + navHeight
            view.requestLayout()
        }
    }

private val actionPaddingNavigation: (View, Int, Int) -> Unit =
    { view, rawBottomPadding, navHeight ->
        view.setPadding(
            view.paddingLeft,
            view.paddingTop,
            view.paddingRight,
            rawBottomPadding + navHeight
        )
    }

fun fixNavBarMargin(vararg views: View) {
    views.forEach {
        fixSingleNavBarMargin(it)
    }
}

private fun fixSingleNavBarMargin(view: View) {
    val lp = view.layoutParams as? ViewGroup.MarginLayoutParams ?: return
    val rawBottomMargin = lp.bottomMargin

    val viewForCalculate = getViewForCalculate(view)

    if (viewForCalculate.isAttachedToWindow) {
        val realNavigationBarHeight = getRealNavigationBarHeight(viewForCalculate)
        lp.bottomMargin = rawBottomMargin + realNavigationBarHeight
        view.requestLayout()
    }
    
    //isAttachedToWindow方法并不能保证此时的WindowInsets是正确的，仍然需要添加监听
    val hostRef = WeakReference(viewForCalculate)
    val viewRef = WeakReference(view)
    val info = NavigationViewInfo(hostRef, viewRef, rawBottomMargin, actionMarginNavigation)
    navigationViewInfoList.add(info)
    viewForCalculate.setOnApplyWindowInsetsListener(onApplyWindowInsetsListener)
}

fun paddingByNavBar(view: View) {
    val rawBottomPadding = view.paddingBottom

    val viewForCalculate = getViewForCalculate(view)

    if (viewForCalculate.isAttachedToWindow) {
        val realNavigationBarHeight = getRealNavigationBarHeight(viewForCalculate)
        view.setPadding(
            view.paddingLeft,
            view.paddingTop,
            view.paddingRight,
            rawBottomPadding + realNavigationBarHeight
        )
    }

    //isAttachedToWindow方法并不能保证此时的WindowInsets是正确的，仍然需要添加监听
    val hostRef = WeakReference(viewForCalculate)
    val viewRef = WeakReference(view)
    val info =
        NavigationViewInfo(hostRef, viewRef, rawBottomPadding, actionPaddingNavigation)
    navigationViewInfoList.add(info)
    viewForCalculate.setOnApplyWindowInsetsListener(onApplyWindowInsetsListener)
}

/**
* Dialog下的View在低版本机型中获取到的WindowInsets值有误，
* 所以尝试去获得Activity的contentView，通过Activity的contentView获取WindowInsets
*/
@SuppressLint("ContextCast")
private fun getViewForCalculate(view: View): View {
    return (view.context as? ContextWrapper)?.let {
        return@let (it.baseContext as? Activity)?.findViewById<View>(android.R.id.content)?.rootView
    } ?: view.rootView
}

/**
* 仅当view attach window后生效
*/
private fun getRealNavigationBarHeight(view: View): Int {
    val insets = ViewCompat.getRootWindowInsets(view)
        ?.getInsets(WindowInsetsCompat.Type.navigationBars())
    return insets?.bottom ?: getNavigationBarHeight(view.context)
}
```

我简单解释一下这段代码：为所有需要沉浸的页面的根`View`设置同一个回调，并将待适配导航栏高度的`View`添加到列表中，当`WindowInsets`回调触发后，遍历这个列表，判断触发回调的`View`的`host`是否与待适配导航栏高度的`View`对应，对应的话则处理`View`适配导航栏高度

这里需要注意，`WindowInsets`的分发其实是在`dispatchAttachedToWindow`之后的，所以`isAttachedToWindow`方法并不能保证此时的`WindowInsets`是正确的，具体可以去看`ViewRootImpl`中的源码，关键方法：`dispatchApplyInsets`，这里判断`isAttachedToWindow`并设置高度是为了防止出现`View`已经完全布局完成，之后再也不会触发`OnApplyWindowInsets`的情况

这里我也测试了内存泄漏情况，确认无内存泄漏，大家可以放心食用

# 底部Dialog适配沉浸式

底部`Dialog`适配沉浸式要比正常的`Activity`更麻烦一些，主要问题也是集中在沉浸式导航栏上

## 获取导航栏高度

仔细的小伙伴们可以已经注意到了我在沉浸式导航栏获取高度那里代码中的注释，`Dialog`下的`View`在低版本机型（经测试，`Android 9`一下就会有这个问题）中获取到的`WindowInsets`值有误，所以尝试去获得`Activity`的`contentView`，通过`Activity`的`contentView`获取`WindowInsets`

## LayoutParams导致的异常

在某些系统上（比如MIUI），当我`window.setLayout(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)`时，沉浸式会出现问题，状态栏会被蒙层盖住，`Dialog`底部的内容也会被一个莫名其妙的东西遮挡住

我的解决方案是，`window.setLayout(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT)`，然后布局最外层全部占满，内部留一个底部容器

```xml
<!-- dialog_pangu_bottom_wrapper -->
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/transparent">

    <FrameLayout
        android:id="@+id/pangu_bottom_dialog_container"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom"
        android:clickable="true"
        android:focusable="true" />

</FrameLayout>
```

然后在代码中重写`setContentView`方法

```kotlin
private var canceledOnTouchOutside = true

override fun setContentView(layoutResID: Int) {
    setContentView(
        LayoutInflater.from(context).inflate(layoutResID, null, false)
    )
}

override fun setContentView(view: View) {
    setContentView(
        view,
        ViewGroup.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.WRAP_CONTENT
        )
    )
}

override fun setContentView(view: View, params: ViewGroup.LayoutParams?) {
    val root =
        LayoutInflater.from(context).inflate(R.layout.dialog_pangu_bottom_wrapper, null, false)
    root.setOnClickListener {
        if (canceledOnTouchOutside) {
            dismiss()
        }
    }
    val container = root.findViewById<ViewGroup>(R.id.pangu_bottom_dialog_container)
    container.addView(view, params)

    super.setContentView(
        root,
        ViewGroup.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.MATCH_PARENT
        )
    )
}

override fun setCanceledOnTouchOutside(cancel: Boolean) {
    super.setCanceledOnTouchOutside(cancel)
    canceledOnTouchOutside = cancel
}
```

这样的话视觉效果就和普通的底部`Dialog`一样了，为了进一步减小底部`Dialog`显示隐藏动画之间的差异，我将动画插值器从`linear_interpolator`换成了`decelerate_interpolator`和`accelerate_interpolator`

```xml
<!-- dialog_enter_from_bottom_to_top -->
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"
    android:fromYDelta="100%"
    android:interpolator="@android:anim/decelerate_interpolator"
    android:toYDelta="0" />
```

```xml
<!-- dialog_exit_from_top_to_bottom -->
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"
    android:fromYDelta="0"
    android:interpolator="@android:anim/accelerate_interpolator"
    android:toYDelta="100%" />
```

# 尾声

自此，目前沉浸式遇到的问题全部都解决了，如果以后发现了什么新的问题，我会在这篇文章中补充说明，如果还有什么不明白的地方可以评论，我考虑要不要拿几个具体的场景实战讲解，各位看官老爷麻烦点个赞收个藏不迷路😄