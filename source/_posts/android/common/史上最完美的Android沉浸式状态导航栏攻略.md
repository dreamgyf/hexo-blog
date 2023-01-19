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

最近我在小破站开发一款新App，叫**高能链**。我是一个完美主义者，所以不管对架构还是UI，我都是比较抠细节的，在状态栏和导航栏沉浸式这一块，我还是踩了挺多坑，费了挺多精力的。这次我将我踩坑，适配各机型总结出来的史上最完美的Android沉浸式状态导航栏攻略分享给大家，大家也可以去[高能链官网](https://www.upowerchain.com/)下载体验一下我们的App，实际感受一下沉浸式状态导航栏的效果（登录，实名等账号相关页面由于不是我开发的，就没有适配沉浸式导航栏啦，嘻嘻）

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