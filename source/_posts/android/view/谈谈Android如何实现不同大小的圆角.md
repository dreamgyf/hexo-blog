---
title: 谈谈Android如何实现不同大小的圆角
date: 2023-07-31 15:32:03
tags: 圆角
categories: Android
---

# 简介

在开发过程中，设计常常会有一些比较炫酷的想法，比如两边不一样大小的圆角啦，甚至四角的`radius`各不相同，对于这种情况我们该怎么实现呢？

# 背景圆角

## Shape

对于一般的背景，我们可以直接使用`shape`，这种方法天生支持设置四角不同的`radius`，比如：

```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <solid android:color="#8358FF" />
    <corners
        android:bottomLeftRadius="10dp"
        android:bottomRightRadius="20dp"
        android:topLeftRadius="30dp"
        android:topRightRadius="40dp" />
</shape>
```

小贴士：`shape`在代码层的实现为`GradientDrawable`，可以直接在代码层构建圆角背景，顺便推荐一下我写的库：[ShapeLayout](https://github.com/dreamgyf/ShapeLayout)，可以方便的实现`shape`背景，告别`xml`

# 内容圆角

很多情况下，设置背景的四边不同圆角并不能满足我们，大多数情况下，我们需要连着里面的内容一起切圆角，这里我们需要先指正一下网上的一个错误写法

有人发文说，可以通过`outline.setConvexPath`方法，实现四角不同`radius`，如下：

```kotlin
outline?.setConvexPath(
    Path().apply {
        addRoundRect(
            0f, 0f, width.toFloat(), height.toFloat(),
            floatArrayOf(
                topLeftRadius,
                topLeftRadius,
                topRightRadius,
                topRightRadius,
                bottomRightRadius,
                bottomRightRadius,
                bottomLeftRadius,
                bottomLeftRadius
            ),
            Path.Direction.CCW
        )
    }
)
```

经过实测，这样写是不行的，准确的来说，在大部分系统上是不行的（MIUI上可以，我不知道是该夸它兼容性太好了还是该吐槽它啥，我的测试机用的小米，这导致我在最后的测试阶段才发现这个问题）

指出错误方法后，让我们来看看正确解法有哪些

## CardView

说到切内容圆角，我们自然而然会去想到`CardView`，其实`CardView`的圆角也是通过`Outline`实现的

有人可能要问了，`CardView`不是只支持四角相同`radius`吗？别急，且看我灵机一动想出来的神奇嵌套大法

### 神奇嵌套大法

既然一个`CardView`只能设一个`radius`，那我多用几个`CardView`嵌套是否能解决问题呢？

举个最简单的例子，比如说设计想要上半部分为`12dp`的圆角，下半部分没有圆角，我们需要一个辅助`View`，让他的顶部和父布局的底部对齐，然后设置成圆角大小的高度或者`margin`，接着使用`CardView`，让它的底部对齐这个辅助`View`的底部，再设置一个圆角大小的`padding`，这样，由于`CardView`超出了父布局的边界，所以底部的圆角不会显示出来，再由于我们设置了恰好的`padding`，所以`CardView`里面的内容也能完整展示，可谓完美，实例如下：

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <Space
        android:id="@+id/guideline"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_marginTop="12dp"
        app:layout_constraintTop_toBottomOf="parent" />

    <androidx.cardview.widget.CardView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:cardBackgroundColor="@android:color/transparent"
        app:cardCornerRadius="12dp"
        app:cardElevation="0dp"
        app:contentPaddingBottom="12dp"
        app:layout_constraintBottom_toBottomOf="@+id/guideline">

        <ImageView
            android:layout_width="match_parent"
            android:layout_height="300dp"
            android:adjustViewBounds="true"
            android:background="#8358FF" />

    </androidx.cardview.widget.CardView>

</androidx.constraintlayout.widget.ConstraintLayout>
```

上面的例子没有嵌套，因为另一边没有圆角，那么如果我们需要上半部分为`12dp`的圆角，下半部分为`6dp`的圆角，我们可以这样操作

手法和上面的例子一样，不过我们在最外层再嵌套一个`CardView`，并且将其圆角设为较小的那个圆角大小`6dp`，将里面的`CardView`的圆角设置成较大的那个圆角大小`12dp`，具体实现如下：

```xml
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:cardBackgroundColor="@android:color/transparent"
    app:cardCornerRadius="6dp"
    app:cardElevation="0dp"
    app:layout_constraintTop_toTopOf="parent">

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <Space
            android:id="@+id/guideline"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_marginTop="12dp"
            app:layout_constraintTop_toBottomOf="parent" />

        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:cardBackgroundColor="@android:color/transparent"
            app:cardCornerRadius="12dp"
            app:cardElevation="0dp"
            app:contentPaddingBottom="12dp"
            app:layout_constraintBottom_toTopOf="@+id/guideline">

            <ImageView
                android:layout_width="match_parent"
                android:layout_height="300dp"
                android:adjustViewBounds="true"
                android:background="#8358FF" />

        </androidx.cardview.widget.CardView>

    </androidx.constraintlayout.widget.ConstraintLayout>

</androidx.cardview.widget.CardView>
```

本质上就是大圆角套小圆角，大圆角的裁切范围更大，会覆盖小圆角裁切的范围，从视觉上看就实现了两边的不同圆角

那么如果我们想进一步实现三边不同圆角或者四边不同圆角呢？原理和上面是一样的，只不过嵌套和占位会变得更加复杂，记住一个原则，小圆角在外，大圆角在内即可，我直接把具体实现贴在下面，各位自取即可：

- 三边不同圆角（左下6dp，左上12dp，右上24dp）

```xml
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <Space
        android:id="@+id/guideline"
        android:layout_width="6dp"
        android:layout_height="match_parent"
        app:layout_constraintStart_toEndOf="parent" />

    <androidx.cardview.widget.CardView
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:cardBackgroundColor="@android:color/transparent"
        app:cardCornerRadius="6dp"
        app:cardElevation="0dp"
        app:contentPaddingRight="6dp"
        app:layout_constraintEnd_toEndOf="@+id/guideline"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent">

        <androidx.constraintlayout.widget.ConstraintLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <Space
                android:id="@+id/guideline2"
                android:layout_width="match_parent"
                android:layout_height="0dp"
                android:layout_marginTop="12dp"
                app:layout_constraintTop_toBottomOf="parent" />

            <androidx.cardview.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                app:cardBackgroundColor="@android:color/transparent"
                app:cardCornerRadius="12dp"
                app:cardElevation="0dp"
                app:contentPaddingBottom="12dp"
                app:layout_constraintBottom_toTopOf="@+id/guideline2">

                <androidx.constraintlayout.widget.ConstraintLayout
                    android:layout_width="match_parent"
                    android:layout_height="match_parent">

                    <Space
                        android:id="@+id/guideline3"
                        android:layout_width="0dp"
                        android:layout_height="match_parent"
                        android:layout_marginEnd="24dp"
                        app:layout_constraintEnd_toStartOf="parent" />

                    <Space
                        android:id="@+id/guideline4"
                        android:layout_width="match_parent"
                        android:layout_height="0dp"
                        android:layout_marginTop="24dp"
                        app:layout_constraintTop_toBottomOf="parent" />

                    <androidx.cardview.widget.CardView
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        app:cardBackgroundColor="@android:color/transparent"
                        app:cardCornerRadius="24dp"
                        app:cardElevation="0dp"
                        app:contentPaddingBottom="24dp"
                        app:contentPaddingLeft="24dp"
                        app:layout_constraintBottom_toBottomOf="@+id/guideline4"
                        app:layout_constraintEnd_toEndOf="parent"
                        app:layout_constraintStart_toStartOf="@+id/guideline3">

                        <ImageView
                            android:layout_width="match_parent"
                            android:layout_height="300dp"
                            android:adjustViewBounds="true"
                            android:background="#8358FF" />

                    </androidx.cardview.widget.CardView>

                </androidx.constraintlayout.widget.ConstraintLayout>

            </androidx.cardview.widget.CardView>

        </androidx.constraintlayout.widget.ConstraintLayout>

    </androidx.cardview.widget.CardView>

</androidx.constraintlayout.widget.ConstraintLayout>
```

- 四边不同圆角（左下6dp，左上12dp，右上24dp，右下48dp）

```xml
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <Space
        android:id="@+id/guideline"
        android:layout_width="6dp"
        android:layout_height="match_parent"
        app:layout_constraintStart_toEndOf="parent" />

    <androidx.cardview.widget.CardView
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        app:cardBackgroundColor="@android:color/transparent"
        app:cardCornerRadius="6dp"
        app:cardElevation="0dp"
        app:contentPaddingRight="6dp"
        app:layout_constraintEnd_toEndOf="@+id/guideline"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent">

        <androidx.constraintlayout.widget.ConstraintLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <Space
                android:id="@+id/guideline2"
                android:layout_width="match_parent"
                android:layout_height="0dp"
                android:layout_marginTop="12dp"
                app:layout_constraintTop_toBottomOf="parent" />

            <androidx.cardview.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                app:cardBackgroundColor="@android:color/transparent"
                app:cardCornerRadius="12dp"
                app:cardElevation="0dp"
                app:contentPaddingBottom="12dp"
                app:layout_constraintBottom_toTopOf="@+id/guideline2">

                <androidx.constraintlayout.widget.ConstraintLayout
                    android:layout_width="match_parent"
                    android:layout_height="match_parent">

                    <Space
                        android:id="@+id/guideline3"
                        android:layout_width="0dp"
                        android:layout_height="match_parent"
                        android:layout_marginEnd="24dp"
                        app:layout_constraintEnd_toStartOf="parent" />

                    <Space
                        android:id="@+id/guideline4"
                        android:layout_width="match_parent"
                        android:layout_height="0dp"
                        android:layout_marginTop="24dp"
                        app:layout_constraintTop_toBottomOf="parent" />

                    <androidx.cardview.widget.CardView
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        app:cardBackgroundColor="@android:color/transparent"
                        app:cardCornerRadius="24dp"
                        app:cardElevation="0dp"
                        app:contentPaddingBottom="24dp"
                        app:contentPaddingLeft="24dp"
                        app:layout_constraintBottom_toBottomOf="@+id/guideline4"
                        app:layout_constraintEnd_toEndOf="parent"
                        app:layout_constraintStart_toStartOf="@+id/guideline3">

                        <androidx.constraintlayout.widget.ConstraintLayout
                            android:layout_width="match_parent"
                            android:layout_height="match_parent">

                            <Space
                                android:id="@+id/guideline5"
                                android:layout_width="0dp"
                                android:layout_height="match_parent"
                                android:layout_marginEnd="48dp"
                                app:layout_constraintEnd_toStartOf="parent" />

                            <Space
                                android:id="@+id/guideline6"
                                android:layout_width="match_parent"
                                android:layout_height="0dp"
                                android:layout_marginBottom="48dp"
                                app:layout_constraintBottom_toTopOf="parent" />

                            <androidx.cardview.widget.CardView
                                android:layout_width="0dp"
                                android:layout_height="wrap_content"
                                app:cardBackgroundColor="@android:color/transparent"
                                app:cardCornerRadius="48dp"
                                app:cardElevation="0dp"
                                app:contentPaddingLeft="48dp"
                                app:contentPaddingTop="48dp"
                                app:layout_constraintEnd_toEndOf="parent"
                                app:layout_constraintStart_toStartOf="@+id/guideline5"
                                app:layout_constraintTop_toTopOf="@+id/guideline6">

                                <ImageView
                                    android:layout_width="match_parent"
                                    android:layout_height="300dp"
                                    android:adjustViewBounds="true"
                                    android:background="#8358FF" />

                            </androidx.cardview.widget.CardView>

                        </androidx.constraintlayout.widget.ConstraintLayout>

                    </androidx.cardview.widget.CardView>

                </androidx.constraintlayout.widget.ConstraintLayout>

            </androidx.cardview.widget.CardView>

        </androidx.constraintlayout.widget.ConstraintLayout>

    </androidx.cardview.widget.CardView>

</androidx.constraintlayout.widget.ConstraintLayout>
```

## 自定义ImageView

由于大部分裁切内容的需求，其中的内容都是图片，所以我们也可以直接对图片进行裁切，此时我们就可以自定义`ImageView`来将图片裁剪出不同大小的圆角

### clipPath

先说这个方法的缺点，那就是无法使用抗锯齿，这一点缺陷注定了它无法被正式使用，但我们还是来看看他是如何实现的

首先，我们需要重写`ImageView`的`onSizeChanged`方法，为我们的`Path`确定路线

```kotlin
override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
    super.onSizeChanged(w, h, oldw, oldh)
    path.reset()
    //这里的radii便是我们自定义的四边圆角大小的数组（size为8，从左上顺时针到左下）
    path.addRoundRect(0f, 0f, w.toFloat(), h.toFloat(), radii, Path.Direction.CW)
}
```

接着我们重写`onDraw`方法

```kotlin
override fun onDraw(canvas: Canvas) {
    canvas.clipPath(path)
    super.onDraw(rawBitmapCanvas)
}
```

网上有的教程说要设置`PaintFlagsDrawFilter`，但实际上就算为这个`PaintFlagsDrawFilter`设置了`Paint.ANTI_ALIAS_FLAG`抗锯齿属性也没用，抗锯齿只在使用了`Paint`的情况下才可以生效

### PorterDuff

既然`clipPath`无法使用抗锯齿，那我们可以换一条路线曲线救国，那就是使用`PorterDuff`

当然，这种方法也有它的缺点，那就是不能使用硬件加速，但相比无法使用抗锯齿而言，这点缺点也就不算什么了

首先，我们要在构造方法中禁用硬件加速

```kotlin
init {
    setLayerType(LAYER_TYPE_SOFTWARE, null)
}
```

然后重写`onSizeChanged`方法，在这个方法中，我们需要确定`Path`，构造出相应大小的`Bitmap`和`Canvas`，这俩是用来获取原始无圆角的`Bitmap`的

```kotlin
override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
    super.onSizeChanged(w, h, oldw, oldh)
    path.reset()
    path.addRoundRect(0f, 0f, w.toFloat(), h.toFloat(), radii, Path.Direction.CW)

    rawBitmap = Bitmap.createBitmap(w, h, Bitmap.Config.ARGB_8888)
    rawBitmapCanvas = Canvas(rawBitmap!!)
}
```

接着我们重写`onDraw`方法

```kotlin
private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
private val xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_IN)

override fun onDraw(canvas: Canvas) {
    val rawBitmap = rawBitmap ?: return
    val rawBitmapCanvas = rawBitmapCanvas ?: return

    super.onDraw(rawBitmapCanvas)

    canvas.drawPath(path, paint)
    paint.xfermode = xfermode
    canvas.drawBitmap(rawBitmap, 0f, 0f, paint)
    paint.xfermode = null
}
```

这里，我们调用父类的`onDraw`方法，获取到原始无圆角的`Bitmap`，然后绘制`Path`，再通过`PorterDuff`的叠加效果绘制我们刚刚得到的原始`Bitmap`，由于`PorterDuff.Mode.SRC_IN`的效果是取两层绘制交集，显示上层，所以我们最终便获得了一个带圆角的图片

# 截图问题

如果想要将`View`截图成`Bitmap`，在`Android 8.0`及以上系统中我们可以使用`PixelCopy`，此时使用`CardView`或`Outline`裁切的圆角不会有任何问题，而在`Android 8.0`以下的系统中，通常我们是构建一个带`Bitmap`的`Canvas`，然后对要截图的`View`调用`draw`方法达成截图效果，而在这种情况下，使用`CardView`或`Outline`裁切的圆角便会出现无效的情况（截图出来的`Bitmap`中，圆角没了），这种情况的出现似乎也和硬件加速有关，针对这种问题，我个人的想法是准备两套布局，`8.0`以上使用`CardView`或`Outline`，截图使用`PixelCopy`，`8.0`以下使用`PorterDuff`方案直接裁切图片，最大程度避免性能损耗

# 总结

以上就是我本人目前对`Android`实现不同大小的圆角的一些想法和遇到的问题，至于`CardView`嵌套会不会带来什么性能问题，我目前并没有做验证，各位小伙伴有什么更好的解决方案，欢迎在评论区指出，大家一起集思广益