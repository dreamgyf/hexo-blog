---
title: 自定义EditView时踩过的坑
date: 2020-10-19 12:05:34
tags: 自定义View
---

# 简介
这次的需求是一个单词拼写的输入框，要求每个字母分割开来输入，每个字母下面有一个下划线，就类似于验证码输入或者支付密码输入的效果
![效果图](/images/spell_edit_text.png "效果图")

最终成品是一个自定义View，实现参考了[VercodeEditText](https://github.com/JingYeoh/VercodeEditText)

---

# EditView还是TextView？

刚开始的时候，我选择了继承`AppCompatEditText`，但在我试着draw on canvas的时候，奇怪的发现绘制的东西并没有在界面上显示，然后我尝试将视图的高度调大，当到达了一个临界点后，内容突然显示出来了。

为了得知这个问题的成因，我试着画一个占满整个画布的矩形，打开开发者选项里的显示布局边界后发现，这个矩形并没有占满整个布局。一开始猜想可能是因为画布的高度小于视图的高度，于是打开debug调试断点发现，两者是一致的；然后猜测是不是画布因为什么原因产生了偏移？但怎么尝试都没有得到确定性的结论，直到某次我打开了EditText的背景，并随便滑动了几下发现，在向下滑动的时候，原本绘制的内容便从视图最上方滚动了下来。原来是因为当EditText高度小于1行时，EditText会自动适配并滚动到最下方。

既然知道了问题的成因，那就开始着手解决他，最直接的办法就是禁止EditText的滚动。为此，我尝试了`setMinHeight()`，`setMinLines()`都没有用。然后我退而求其次，尝试使用`scrollTo(0, 0)`，将视图固定滑动到最顶部，发现效果并不是很理想。然后在查资料的过程中我发现了`MovementMethod`这么一个东西。

网上关于`MovementMethod`的资料比较少，我查询了一下Google的官方文档里面介绍：
> Provides cursor positioning, scrolling and text selection functionality in a TextView.
> 即：在 TextView提供了光标定位，滚动和文本选择功能。

找到了产生滚动的元凶，那问题就好办了，在源码里可以看到，EditText的`getDefaultMovementMethod()`返回了一个`ArrowKeyMovementMethod`，我们直接`setMovementMethod(null)`或者重写父类的`getDefaultMovementMethod()`使其返回值为null，滚动的问题便解决了。

解决完这一步后，又发现了一个新问题，EditText的上下左右有一定的padding，点击到这部分padding的区域是不会触发EditText的获取焦点弹出输入法的，当然也可以直接重写`onTouchEvent`方法加上`requestFocus()`方法解决，但考虑到继承EditText要重设背景，又要setMovementMethod，还要处理边缘点击事件，感觉太麻烦，不如直接继承TextView，处理的事情会稍微少一些。

于是我选择继承`AppCompatTextView`，重写`getDefaultEditable()`使其返回true以打开编辑功能，`setFocusableInTouchMode(true)`使其能获取焦点，重写`onTouchEvent`方法加上`requestFocus()`方法使其点击能够直接获取焦点，`setCursorVisible(false)`隐藏光标，`setLongClickable(false)`禁止长按弹出编辑菜单，到这一步，基本难点已经解决了。

---

# onTextChanged多次调用？

为了监听文本改变的事件，我一开始选择了自定义View直接implements TextWatcher，然后`addTextChangedListener(this)`，这样在断点调试的时候发现，`onTextChanged()`方法被执行了多次，但`beforeTextChanged()`和`afterTextChanged()`执行次数却是正常的。原来，在TextView内部已经有了一个可重写的`onTextChanged()`方法，和TextWatcher里的`onTextChanged()`一模一样，当`addTextChangedListener(this)`后，TextView会先执行TextWatcher的`onTextChanged()`，再执行自己内部的`onTextChanged()`。解决方法很简单，将implements TextWatcher去掉，改为addTextChangedListener一个匿名内部类就好了。

---

# 长度超过限制了怎么办？

刚开始，我在`onTextChanged()`里增加了对Text长度的判断，如果长度超长，就把原Text截断到最大长度，然后重新setText进去。这样做有一个问题，这样并不能保证`afterTextChanged()`回调里的Text参数长度合法。当这么做后，TextView首先会触发截断Text的`afterTextChanged()`，然后再触发超长Text的`afterTextChanged()`。后来在搜索资料的时候发现，TextView内部持有了一个`InputFilter`数组，这个接口可以很好的帮助我们在触发回调之前对输入的字符串进行过滤操作。

- InputFilter接口方法

> public CharSequence filter(CharSequence source, int start, int end,Spanned dest, int dstart, int dend);

其中，source参数便是在用户输入之后，触发回调之前的字符串，我们可以对他进行处理，返回处理后的字符串来满足长度限制的需求。

```java
public CharSequence filter(CharSequence source, int start, int end, Spanned dest, int dstart, int dend) {
    if (source.length() > getLength()) {
		return source.subSequence(0, getLength());
	}
	return source;
}
```

# 具体的绘制实现？

## 计算预留位置

我定义了一个变量`String ALL_CHARS = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"`用来计算用户输入的预留宽高。

```java
private void measureChar() {
	int width = 0, height = 0;
	Rect rect = new Rect();
	for (int i = 0; i < ALL_CHARS.length(); i++) {
		mTextPaint.getTextBounds(ALL_CHARS, i, i + 1, rect);
		width = Math.max(width, rect.width());
		height = Math.max(height, rect.height());
	}

	mCharWidth = width;
	mCharHeight = height;
}
```

在onMeasure()里计算好布局宽高后，根据宽高和间隔距离平均分配一下每个字母的坐标。

## 自适应宽度

这次的需求还要求当布局的宽度超过父布局时，自动缩小字体大小以适应父布局宽度。这个其实也很简单，计算一下绘制需要的宽度，如果超过父布局宽度，就减小字号，循环一下即可。

```java
int answerWidth = mCharWidth * getLength() + mSpacingPx * (getLength() - 1);
while (answerWidth > width - getPaddingLeft() - getPaddingRight()) {
	mTextPaint.setTextSize(--mTextSize);
	measureChar();
	answerWidth = mCharWidth * getLength() + mSpacingPx * (getLength() - 1);
}
```

## 用户输入文字的绘制

由于每个字母的宽高可能不同，所以不能直接使用之前计算好的坐标绘制，需要使用之前测量好的预留的宽度减去用户实际输入字母的宽度除以2，然后加上这个预留位置的起始坐标。

```java
private float computeCharX(CharCoordinate coordinate, char letter) {
	mTextPaint.getTextBounds(String.valueOf(letter), 0, 1, mTempRect);
	int realCharWidth = mTempRect.width();
	return coordinate.start + (float) (mCharWidth - realCharWidth) / 2 - mTempRect.left;
}
```
> 这里减去mTempRect.left是因为绘制出来的字符有些向右偏离

## 绘制光标

TextView原本的光标不符合我们的需求，我们需要绘制一下自定义的光标。

先定义一下光标闪烁时间：

```java
private final static int DEFAULT_CURSOR_DURATION = 800;
private int mCursorDuration = DEFAULT_CURSOR_DURATION;
```

再定义一个Handler和Runnable用来间隔执行任务

```java
private Runnable mCursorRunnable = new Runnable() {
	@Override
	public void run() {
		if (mNeedCursorShow) {
			mIsCursorShowing = !mIsCursorShowing;
			invalidate();
		}
		mHandler.postDelayed(mCursorRunnable, mCursorDuration);
	}
};
```

这样通过设置一个bool值和定时任务每隔一段时间刷新一下视图就可以轻松实现光标的闪烁。