---
title: 走马灯式横向滚动的TextView
date: 2021-05-26 18:01:00
tags: TextView
categories: Android
---

# 简介

我们可以设置`TextView`的`android:ellipsize="marquee"`属性，来做到当文字超出一行的时候呈现跑马灯效果。但`TextView`的这个走马灯效果需要获取焦点，而同一时间只有一个控件可以获得焦点，更重要的是产品要求无论文字内容是否超出一行，都要滚动效果。

这里先贴一下最后实现的Github地址和效果图

[https://github.com/dreamgyf/MarqueeTextView](https://github.com/dreamgyf/MarqueeTextView)

![MarqueeTextView](https://camo.githubusercontent.com/f78ec92d9270fe6a72f182090567334a5d9ecb5221f471a34cbd83905be65c6a/68747470733a2f2f647265616d6779662d636f64696e672e6f73732d636e2d7368616e676861692e616c6979756e63732e636f6d2f4d61727175656554657874566965772f4d61727175656554657874566965772e676966)

# 思路

思路其实很简单，我们只要将单行的`TextView`截成一张`Bitmap`，然后我们再自定义一个View，重写它的`onDraw`方法，每隔一段时间，将这张Bitmap画在不同的坐标上（左右两边各draw一次），这样连续起来看起来就是走马灯效果了。

后来和同事讨论，他提出能不能通过Canvas的平移配合`drawText`实现这个功能，我想应该也是可以的，但我没有做尝试，各位看官感兴趣的可是试一下这种方案。

# 实现

我们先自定义一个View继承自`AppCompatTextView`，再在初始化的时候new一个`TextView`，并重写`onMeasure`和`onLayout`方法

```java
private void init() {
    mTextView = new TextView(getContext(), attrs);
    //TextView如果没有设置LayoutParams，当setText的时候会引发NPE导致崩溃
    mTextView.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT));
    mTextView.setMaxLines(1);
}

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    //宽度不设限制
    mTextView.measure(MeasureSpec.UNSPECIFIED, heightMeasureSpec);
}

@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
    //保证布局包含完整的Text内容
    mTextView.layout(left, top, left + mTextView.getMeasuredWidth(), bottom);
}
```

这样做是为了利用这个内部`TextView`生成我们需要的`Bitmap`，同时借用`TextView`写好的`onMeasure`方法，这样我们就不用再那么复杂的重写`onMeasure`方法了

接下来是生成`Bitmap`

```java
private void updateBitmap() {
    mBitmap = Bitmap.createBitmap(mTextView.getMeasuredWidth(), getMeasuredHeight(), Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(mBitmap);
    mTextView.draw(canvas);
}
```

这个很简单，需要注意的是长度要使用内部持有的`TextView`的`getMeasuredWidth`，如果使用`getWidth`的话，最大值为屏幕的宽度，很可能导致生成出的`Bitmap`不全，高度用谁的倒是无所谓

在每次`setText`或`setTextSize`的时候都需要更新`Bitmap`并重新布局绘制

```java
private void init() {
    mTextView.addOnLayoutChangeListener(new OnLayoutChangeListener() {
        @Override
        public void onLayoutChange(View v, int left, int top, int right, int bottom, int oldLeft, int oldTop, int oldRight, int oldBottom) {
            updateBitmap();
            restartScroll();
        }
    });
}

@Override
public void setText(CharSequence text, BufferType type) {
    super.setText(text, type);
    //执行父类构造函数时，如果AttributeSet中有text参数会先调用setText，此时mTextView尚未初始化
    if (mTextView != null) {
        mTextView.setText(text);
        requestLayout();
    }
}

@Override
public void setTextSize(int unit, float size) {
    super.setTextSize(unit, size);
    //执行父类构造函数时，如果AttributeSet中有textSize参数会先调用setTextSize，此时mTextView尚未初始化
    if (mTextView != null) {
        mTextView.setTextSize(size);
        requestLayout();
    }
}
```

接下来，我给这个`MarqueeTextView`定义了一些参数，一个是`space`（文字滚动时，头尾的最小间隔距离），另一个是`speed`（文字滚动的速度）

先看一下`onDraw`的实现吧

```java
@Override
protected void onDraw(Canvas canvas) {
    if (mBitmap != null) {
        //当文字内容不超过一行
        if (mTextView.getMeasuredWidth() <= getWidth()) {
            //计算头尾需要间隔的宽度
            int space = mSpace - (getWidth() - mTextView.getMeasuredWidth());
            if (space < 0) {
                space = 0;
            }

            //当左边的drawBitmap的坐标超过了显示宽度+间隔宽度，即走完一个循环，右边的Bitmap已经挪到了最左边，将坐标重置
            if (mLeftX < -getWidth() - space) {
                mLeftX += getWidth() + space;
            }

            //画左边的bitmap
            canvas.drawBitmap(mBitmap, mLeftX, 0, getPaint());
            if (mLeftX < 0) {
                //画右边的bitmap，位置为最右边的坐标-左边bitmap已消失的宽度+间隔宽度
                canvas.drawBitmap(mBitmap, getWidth() + mLeftX + space, 0, getPaint());
            }
        } else {
            //当文字内容超过一行
            //当左边的drawBitmap的坐标超过了内容宽度+间隔宽度，即走完一个循环，右边的Bitmap已经挪到了最左边，将坐标重置
            if (mLeftX < -mTextView.getMeasuredWidth() - mSpace) {
                mLeftX += mTextView.getMeasuredWidth() + mSpace;
            }

            //画左边的bitmap
            canvas.drawBitmap(mBitmap, mLeftX, 0, getPaint());
            //当尾部已经显示出来的时候
            if (mLeftX + (mTextView.getMeasuredWidth() - getWidth()) < 0) {
                //画右边的bitmap，位置为尾部的坐标+间隔宽度
                canvas.drawBitmap(mBitmap, mTextView.getMeasuredWidth() + mLeftX + mSpace, 0, getPaint());
            }
        }
    }
}
```

这就是基本的绘制思路

接下来需要让他动起来，这里使用的`Choreographer`，每次收到`Vsync`信号系统绘制新帧时都更新一下坐标并重绘

```java
private static final float BASE_FPS = 60f;

private float mFps = BASE_FPS;

/**
 * 获取当前屏幕刷新率
 */
private void updateFps() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        mFps = context.getDisplay().getRefreshRate();
    } else {
        WindowManager windowManager =
                (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        mFps = windowManager.getDefaultDisplay().getRefreshRate();
    }
}

private Choreographer.FrameCallback frameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        invalidate();
        //保证在不同刷新率的屏幕上，视觉上的速度一致
        int speed = (int) (BASE_FPS / mFps * mSpeed);
        mLeftX -= speed;
        Choreographer.getInstance().postFrameCallback(this);
    }
};

public void startScroll() {
    Choreographer.getInstance().postFrameCallback(frameCallback);
}

public void pauseScroll() {
    Choreographer.getInstance().removeFrameCallback(frameCallback);
}

public void stopScroll() {
    mLeftX = 0;
    Choreographer.getInstance().removeFrameCallback(frameCallback);
}

public void restartScroll() {
    stopScroll();
    startScroll();
}
```

最后，在`View`可见性发生变化时，需要控制一下动画的启停

```java
@Override
protected void onVisibilityChanged(@NonNull View changedView, int visibility) {
    if (visibility == VISIBLE) {
        updateFps();
        Choreographer.getInstance().postFrameCallback(frameCallback);
    } else {
        Choreographer.getInstance().removeFrameCallback(frameCallback);
    }
}
```
