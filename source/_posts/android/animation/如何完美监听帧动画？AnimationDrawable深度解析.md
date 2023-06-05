---
title: 如何完美监听帧动画？AnimationDrawable深度解析
date: 2023-06-01 15:20:51
tags: 
- 动画
- AnimationDrawable
categories: 
- [Android, 动画]
---

# 简介

作为苦逼的程序员，产品和设计提出来的需求咱也没法拒绝，这不，前两天设计就给提了个需求，要求在帧动画结束后，把原位置的动画替换成一段文字。我们知道，在`Android`中，帧动画的实现类为`AnimationDrawable`，而这玩意儿又不像`Animator`一样可以通过`addListener`之类的方法监听动画的开始、结束等事件，那我们该怎么监听`AnimationDrawable`的结束事件呢？

目前网上大多数的做法都是获取帧动画的总时长，然后用`Handler`做一个`postDelayed`执行结束后的事情。这种方法怎么说呢？能用，但是不够精准也不够优雅，本文我们将从源码层面解析`AnimationDrawable`是如何将一帧帧的图片组合起来展示成连续的动画的，再从中寻求动画监听的切入点。

**注：只想看实现的朋友们可以直接跳到 包装Drawable.Callback 这一节看最终实现**

# ImageView如何展示Drawable

`AnimationDrawable`说到底它也就是个`Drawable`，而我们一般都是使用`ImageView`作为`Drawable`展示的布局，那我们就以此作为入口开始分析`Drawable`在`ImageView`中是如何被展示的。

回想一下，我们想要给一个`ImageView`设置图片一般可以用下面几种方法：

- `setImageBitmap`
- `setImageResource`
- `setImageURI`
- `setImageDrawable`

`setImageBitmap`会将`Bitmap`包装成一个`BitmapDrawable`，然后再调用`setImageDrawable`方法。

`setImageResource`和`setImageURI`方法会通过`resolveUri`方法从`Resource`或`Uri`中解析出`Drawable`，然后调用`updateDrawable`方法

`setImageDrawable`方法则会直接调用`updateDrawable`方法

最终殊途同归走到`updateDrawable`方法中

```java
private void updateDrawable(Drawable d) {
    ...
    if (mDrawable != null) {
        sameDrawable = mDrawable == d;
        mDrawable.setCallback(null);
        unscheduleDrawable(mDrawable);
        ...
    }

    mDrawable = d;

    if (d != null) {
        d.setCallback(this);
        ...
    } else {
        ...
    }
}
```

可以看到，这里将我们设置的图片资源赋值到`mDrawable`上。注意，这里有一个`Drawable`动起来的关键点，同时也是我们动画监听的最终切入点：`Drawable.setCallback(this)`，我们后面分析帧切换的时候会详细去聊它。

我们知道，一个控件想要绘制内容得在`onDraw`方法中操作`Canvas`，所以让我们再来看看`onDraw`方法
```java
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    if (mDrawable == null) {
        return; // couldn't resolve the URI
    }

    if (mDrawableWidth == 0 || mDrawableHeight == 0) {
        return;     // nothing to draw (empty bounds)
    }

    ...
    mDrawable.draw(canvas);
    ...
}
```

可以看到，这里调用了`Drawable.draw`方法将`Drawable`自身绘制到`ImageView`的`Canvas`上

# DrawableContainer

查看`AnimationDrawable`的继承关系我们可以得知它继承自`DrawableContainer`，从命名中我们就能看出来，它是`Drawable`的容器，我们来看一下它所实现的`draw`方法：

```java
public void draw(Canvas canvas) {
    if (mCurrDrawable != null) {
        mCurrDrawable.draw(canvas);
    }
    if (mLastDrawable != null) {
        mLastDrawable.draw(canvas);
    }
}
```

`mLastDrawable`是为了完成动画的切换效果（出入场动画）所准备的，我们可以不用关心它。

我们可以发现，它的内部有一个名为`mCurrDrawable`的成员变量，我们可以合理猜测它是通过切换`mCurrDrawable`指向的目标`Drawable`来完成展示不同图片的功能，那么事实是这样吗？

没错，`DrawableContainer`给我们提供了一个`selectDrawable`方法，用来切换不同的图片：

```java
public boolean selectDrawable(int index) {
    if (index == mCurIndex) {
        return false;
    }

    ...

    if (index >= 0 && index < mDrawableContainerState.mNumChildren) {
        final Drawable d = mDrawableContainerState.getChild(index);
        mCurrDrawable = d;
        mCurIndex = index;
        ...
    } else {
        mCurrDrawable = null;
        mCurIndex = -1;
    }

    ...

    invalidateSelf();

    return true;
}
```

可以看到，和我们猜想的一样，在`DrawableContainer`的内部有一个子类`DrawableContainerState`用于保存所有的`Drawable`，它继承自`Drawable.ConstantState`，是用来储存`Drawable`间的常量状态和数据的。在`DrawableContainerState`中有一个`mDrawables`数组用于保存所有的`Drawable`，通过`addChild`方法将`Drawable`加入到这个数组中

而在`selectDrawable`方法中，它通过`getChild`方法去获取当前应该显示的`Drawable`，并将其和`index`分别赋值给它的两个成员变量`mCurrDrawable`和`mCurIndex`，然后调用`invalidateSelf`方法执行重绘：

```java
public void invalidateSelf() {
    final Callback callback = getCallback();
    if (callback != null) {
        callback.invalidateDrawable(this);
    }
}
```

`invalidateSelf`被定义实现在`Drawable`类中，还记得我之前让大家注意的`Callback`吗？在设置图片这一步时，它就被赋值了，实际上这个接口被`View`所实现，所以在前面我们可以看到调用`setCallback`时，我们传入的参数为`this`

不过`ImageView`在继承`View`的同时也重写了这个`invalidateDrawable`方法，最终调用了`invalidate`方法执行重绘，此时，一张新的图片就被展示到我们的屏幕上了

```java
//ImageView.invalidateDrawable
public void invalidateDrawable(@NonNull Drawable dr) {
    if (dr == mDrawable) {
        if (dr != null) {
            // update cached drawable dimensions if they've changed
            final int w = dr.getIntrinsicWidth();
            final int h = dr.getIntrinsicHeight();
            if (w != mDrawableWidth || h != mDrawableHeight) {
                mDrawableWidth = w;
                mDrawableHeight = h;
                // updates the matrix, which is dependent on the bounds
                configureBounds();
            }
        }
        /* we invalidate the whole view in this case because it's very
            * hard to know where the drawable actually is. This is made
            * complicated because of the offsets and transformations that
            * can be applied. In theory we could get the drawable's bounds
            * and run them through the transformation and offsets, but this
            * is probably not worth the effort.
            */
        invalidate();
    } else {
        super.invalidateDrawable(dr);
    }
}
```

# AnimationDrawable

`DrawableContainer`分析完后，我们可以很自然的想到，`AnimationDrawable`就是通过`DrawableContainer`这种可以切换图片的机制，每隔一定时间执行一下`selectDrawable`便可以达成帧动画的效果了。

我们先回想一下，在代码中怎么构造出一个多帧的`AnimationDrawable`？没错，用默认构造方法实例化出来后，调用它的`addFrame`方法往里一帧帧的添加图片：

```java
public void addFrame(@NonNull Drawable frame, int duration) {
    mAnimationState.addFrame(frame, duration);
    if (!mRunning) {
        setFrame(0, true, false);
    }
}
```

可以看到`AnimationDrawable`也有一个内部类`AnimationState`，继承自`DrawableContainerState`，它的`addFrame`方法就是调用`DrawableContainerState.addChild`方法添加图片，同时将这张图片的持续时间保存在`mDurations`数组中：

```java
public void addFrame(Drawable dr, int dur) {
    int pos = super.addChild(dr);
    mDurations[pos] = dur;
}
```

想让`AnimationDrawable`动起来的话，我们得要调用它的`start`方法，那我们就从这个方法开始分析：

```java
public void start() {
    mAnimating = true;

    if (!isRunning()) {
        // Start from 0th frame.
        setFrame(0, false, mAnimationState.getChildCount() > 1
                || !mAnimationState.mOneShot);
    }
}
```

这里将`mAnimating`状态置为`true`，然后调用`setFrame`方法从第0帧开始展示图片

```java
private void setFrame(int frame, boolean unschedule, boolean animate) {
    if (frame >= mAnimationState.getChildCount()) {
        return;
    }
    mAnimating = animate;
    mCurFrame = frame;
    selectDrawable(frame);
    if (unschedule || animate) {
        unscheduleSelf(this);
    }
    if (animate) {
        // Unscheduling may have clobbered these values; restore them
        mCurFrame = frame;
        mRunning = true;
        scheduleSelf(this, SystemClock.uptimeMillis() + mAnimationState.mDurations[frame]);
    }
}
```

这里可以看到，和我们所想的一样，调用了`DrawableContainer.selectDrawable`切换当前展示图片，由于我们之前将`mAnimating`赋值为了`true`，所以会调用`scheduleSelf`方法调度展示下一张图片，时间为当前帧持续时间后

```java
public void scheduleSelf(@NonNull Runnable what, long when) {
    final Callback callback = getCallback();
    if (callback != null) {
        callback.scheduleDrawable(this, what, when);
    }
}
```

`scheduleSelf`方法调用了`Drawable.Callback.scheduleDrawable`方法，我们去`View`里面看实现：

```java
public void scheduleDrawable(@NonNull Drawable who, @NonNull Runnable what, long when) {
    if (verifyDrawable(who) && what != null) {
        final long delay = when - SystemClock.uptimeMillis();
        if (mAttachInfo != null) {
            mAttachInfo.mViewRootImpl.mChoreographer.postCallbackDelayed(
                    Choreographer.CALLBACK_ANIMATION, what, who,
                    Choreographer.subtractFrameDelay(delay));
        } else {
            // Postpone the runnable until we know
            // on which thread it needs to run.
            getRunQueue().postDelayed(what, delay);
        }
    }
}
```

实际上两个分支最终都是通过`Handler`实现延时调用，而调用的`Runnable`对象就是之前`scheduleSelf`传入的`this`。没错，`AnimationDrawable`实现了`Runnable`接口：

```java
public void run() {
    nextFrame(false);
}

private void nextFrame(boolean unschedule) {
    int nextFrame = mCurFrame + 1;
    final int numFrames = mAnimationState.getChildCount();
    final boolean isLastFrame = mAnimationState.mOneShot && nextFrame >= (numFrames - 1);

    // Loop if necessary. One-shot animations should never hit this case.
    if (!mAnimationState.mOneShot && nextFrame >= numFrames) {
        nextFrame = 0;
    }

    setFrame(nextFrame, unschedule, !isLastFrame);
}
```

可以看到，在一帧持续时间结束后，便会调用`nextFrame`方法，计算下一帧的`index`，然后调用`setFrame`方法切换下一帧，形成一个循环，这样一帧帧的图片便动了起来，形成了帧动画

# 包装Drawable.Callback

我们从源码层面分析了帧动画是如何运作的，那么怎么监听动画事件相信各位应该都能得出结论了吧？没错，就是重设`Drawable`的`Callback`

当`Drawable`被设置到控件中后，控件会将自身作为`Drawable.Callback`设置给`Drawable`，那么我们只需要重新给`Drawable`设置一个`Drawable.Callback`，在其中调用`View`回调方法的同时，加入自己的监听逻辑即可

```kotlin
val animDrawable = imageView.drawable as AnimationDrawable
val callback = object : Drawable.Callback {
    override fun invalidateDrawable(who: Drawable) {
        imageView.invalidateDrawable(who)
        if (animDrawable.getFrame(animDrawable.numberOfFrames - 1) == current 
                && animDrawable.isOneShot 
                && animDrawable.isRunning 
                && animDrawable.isVisible
        ) {
            val lastFrameDuration = getDuration(animDrawable.numberOfFrames - 1)
            postDelayed({ ...//结束后需要做的事 }, lastFrameDuration.toLong())
        }
    }

    override fun scheduleDrawable(who: Drawable, what: Runnable, `when`: Long) {
        imageView.scheduleDrawable(who, what, `when`)
    }

    override fun unscheduleDrawable(who: Drawable, what: Runnable) {
        imageView.unscheduleDrawable(who, what)
    }
}
//注意一定需要用一个成员变量或其他方式持有这个Callback
//因为Drawable.Callback是以弱引用的形式被保存在Drawable内的，很容易被回收
mCallbackHolder = callback
animDrawable.callback = callback
animDrawable.start()
```

以上的代码便是示例，当满足动画运行到最后一帧，且满足结束状态时，在最后一帧的持续时间后处理结束后需要做的事

当`AnimationDrawable`切换`Visible`状态为`false`时，动画会被暂停，如果在动画结束后触发`setVisible(false)`事件，也会触发`invalidateDrawable`回调，所以这里需要额外判断一下`isVisible`

自己包装的`Drawable.Callback`一定需要找个东西将它强引用起来，因为`Drawable.Callback`是以弱引用的形式被保存在`Drawable`内的，很容易被回收，一旦被回收，整个`AnimationDrawable`动画就动不起来了

# 尾声

为了这么简单一个小功能，还得跑到源码里看怎么实现，对此我的感受是：一入安卓深似海，从此头发是路人