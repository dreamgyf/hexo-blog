---
title: Jetpack Compose入门
date: 2022-04-07 16:54:00
tags: Jetpack Compose
categories: 
- [Android, Jetpack Compose]
---

# 简介

`Jetpack Compose`是用于构建原生`Android`界面的新工具包。它是一种声明式的UI布局，其官方声称可简化并加快`Android`上的界面开发，使用更少的代码、强大的工具和直观的`Kotlin API`，快速让应用生动而精彩。

官网：<https://developer.android.com/jetpack/compose?hl=zh-cn>

我这里也写了一个`Compose`的`Demo`，可以对照着看：<https://github.com/dreamgyf/ComposeDemo>

这个`Demo`实现了：

- `Compose`替代传统布局z

![Compose替代传统布局](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53cc2274f87240558909120b06ac4255~tplv-k3u1fbpfcp-watermark.image?)

- 网格列表效果，类似于传统布局中的`RecyclerView`配合`GridLayoutManager`

![网格列表](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8eb90d2527848958fbe5ae814e5bbfe~tplv-k3u1fbpfcp-watermark.image?)

- 在传统View中使用Compose

- 在Compose中使用传统View

- 自定义布局

# 前置工作

使用`Jetpack Compose`需要先引入一些依赖：

```gradle
dependencies {
    implementation 'androidx.core:core-ktx:1.7.0'
    implementation "androidx.compose.ui:ui:$compose_version"
    implementation "androidx.compose.material:material:$compose_version"
    implementation "androidx.compose.ui:ui-tooling-preview:$compose_version"
    implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.3.1'
    implementation 'androidx.activity:activity-compose:1.3.1'
    debugImplementation "androidx.compose.ui:ui-tooling:$compose_version"
	//网络图片加载三方库
    implementation "io.coil-kt:coil-compose:1.4.0"
}
```

# 可组合函数

`Jetpack Compose`是围绕着可组合函数构建起来的，这些函数以程序化方式定义应用的界面，只需描述应用界面的外观并提供数据依赖项，而不必关注界面的构建过程。此类函数有几个要点：

- 所有可组合函数必须使用`@Composable`注解修饰
- 可组合函数可以像正常函数一样接受参数

```kt
@Composable
fun Demo(name: String) {
    Text(text = "Hello, ${name}!")
}
```

- 可组合函数内部可以书写正常代码（譬如可以通过`if else`控制显示的控件）

```kt
@Composable
fun Demo(showPic: Boolean) {
    if (showPic) {
    	Image(
            painter = painterResource(id = R.drawable.demo),
            contentDescription = null
        )
    }
    Text(text = "Hello, compose!")
}
```

# 单位

`Android`常用的单位`dp`，`sp`等，在`Compose`中以类的形式被定义，使用的方式也很简单，`Compose`借助了`kotlin`的扩展属性，扩展了`Int`，`Double`，`Float`三个基础类，使用方式如下：

```kt
//dp
1.dp; 2.3f.dp; 4.5.dp
//sp
1.sp; 2.3f.sp; 4.5.sp
```

# 资源

如何在`Compose`中使用资源呢，可以通过`xxxResource`方法

```kt
//图片资源
fun painterResource(@DrawableRes id: Int): Painter
//尺寸资源
fun dimensionResource(@DimenRes id: Int): Dp
//颜色资源
fun colorResource(@ColorRes id: Int): Color
//字符串资源
fun stringResource(@StringRes id: Int): String
//字体资源
fun fontResource(fontFamily: FontFamily): Typeface
```

# Modifier

`Modifier`是`Compose`中的布局修饰符，它控制了布局的大小，padding，对齐，背景，边框，裁切，点击等属性，几乎所有的`Compose`布局都需求这项参数，是`Compose`布局中的重中之重

这里介绍一些常用的基本属性，文中没列到的属性可以去官网查看：<https://developer.android.com/jetpack/compose/modifiers-list?hl=zh-cn>

## 尺寸

- `fillMaxWidth`和`fillMaxHeight`相当于`xml`布局中的`match_parent`
- `fillMaxSize`相当于同时设置了`fillMaxWidth`和`fillMaxHeight`
- `wrapContentWidth`和`wrapContentHeight`相当于`xml`布局中的`wrapContent`
- `wrapContentSize`相当于同时设置了`wrapContentWidth`和`wrapContentHeight`
- `width`和`height`则是设置固定宽高，单位为`Dp`
- `size`相当于同时设置了`width`和`height`
- `weight`属性仅在`Row`或`Column`的内部作用域中可以使用，相当于传统`LinearLayout`布局中的`weight`属性

## padding

`padding`方法有几个重载，这些`API`很简单，看参数就很容易能明白意思

## 对齐

`align`属性，使控件可以在父布局中以一种方式对齐，相当于`xml`布局中的layout_gravity属性。另外还有`alignBy`以及`alignByBaseline`属性可以自行研究

## 绘图

- `background`设置背景，不过不能设置图片，如果想以图片作为背景可以使用`Box`布局，在底部垫一个`Image`控件
- `alpha`设置透明度
- `clip`裁剪内容，这个功能很强大，可以直接将视图裁出圆角，圆形等形状

## 操作

- `clickable`方法，可以设置控件的点击事件回调
- `combinedClickable`方法，可以设置控件的点击、双击、长按事件回调
- `selectable`方法，将控件配置为可点击，同时可以设置点击事件

## 滚动

- `horizontalScroll`：使控件支持水平滚动
- `verticalScroll`：使控件支持垂直滚动

## 注意事项

在`Modifier`中设置属性的前后顺序是很重要的，譬如想要一个背景为蓝色的圆角布局，需要先设置`clip`，再设置`background`，反过来`background`会超出圆角范围

# Spacer

`Compose`中没有了`margin`的概念，可以用`Spacer`替代，`Spacer`为留白的意思，使用起来也很简单

```kt
//水平间隔8dp
Spacer(modifier = Modifier.width(8.dp))
```

# 基础布局

## Row & Column

这是两个基本布局组件，其中`Row`为水平布局，`Column`为垂直布局，他们俩接受的参数相似，其中两个参数为`horizontalArrangement`和`verticalAlignment`，他们一个表示水平布局方式，一个表示垂直布局方式，他们默认值为`START`和`TOP`，这两个参数用起来就和传统布局的`gravity`参数一样

## Box

`Box`也是一种基本布局组件，`Box`布局中的组件是可以叠加的，类似传统布局中的`FrameLayout`，可以通过`contentAlignment`参数调整叠加的方式，其默认值为`TopStart`，叠加到左上角，这个参数也和`FrameLayout`的`gravity`参数一样

# 基础控件

## Text

文本控件，对应传统控件TextView，它有以下一些属性

| 属性            | 说明           |
| ------------- | ------------ |
| text          | 文本内容         |
| color         | 文字颜色         |
| fontSize      | 文字大小         |
| fontStyle     | 文本样式（可以设置斜体） |
| fontWeight    | 字重（粗体等）      |
| fontFamily    | 字体           |
| letterSpacing | 文字间距         |
| textAlign     | 文本对齐方式       |
| lineHeight    | 行高           |
| maxLines      | 最大行数         |
| ...           | ...          |

## Image

图片控件，对应传统控件ImageView，它有以下一些属性

| 属性                 | 说明                   |
| ------------------ | -------------------- |
| painter            | 图片内容                 |
| contentDescription | 无障碍描述（可为null）        |
| alignment          | 对齐方式                 |
| contentScale       | 缩放方式（和scaleType属性类似） |
| alpha              | 透明度                  |
| ...                | ...                  |

在开发中经常会面对从网络价值图片的情况，这时候可以借助一些第三方库来解决，这里以coil库为例：

1.  先添加依赖

```gradle
implementation "io.coil-kt:coil-compose:1.4.0"
```

2.  使用

```kt
Image(
	modifier = Modifier
		.size(68.dp, 68.dp)
		.clip(RoundedCornerShape(6.dp)),
	contentScale = ContentScale.Crop,
    painter = rememberImagePainter(picUrl), //使用rememberImagePainter方法填入图片url
    contentDescription = null
)
```

# 列表

`Compose`有两种组件`LazyRow`和`LazyColumn`，一种水平，一种垂直，对应着传统UI中的`RecyclerView`，用这些组件可以方便的构建列表视图，它们需要提供一个`LazyListScope.()`块描述列表项内容

`LazyListScope`的`DSL`提供了多种函数来描述列表项：

```kt
//用于添加单个列表项
fun item(key: Any? = null, content: @Composable LazyItemScope.() -> Unit)
//用于添加多个列表项
fun items(
        count: Int,
        key: ((index: Int) -> Any)? = null,
        itemContent: @Composable LazyItemScope.(index: Int) -> Unit
    )
//用于添加多个列表项
fun <T> LazyListScope.items(
    items: List<T>,
    noinline key: ((item: T) -> Any)? = null,
    crossinline itemContent: @Composable LazyItemScope.(item: T) -> Unit
)
```

示例：

```kt
val list = mutableListOf(0, 1, 2, 3, 4)

LazyColumn {
    //增加单个列表项
    item {
        Text(text = "First item")
    }

    //增加5个列表项
    items(5) { index ->
        Text(text = "Item: $index")
    }
    
    //增加5个列表项
    items(list) { listItem ->
		Text(text = "Item: $listItem")
    }

    //增加单个列表项
    item {
        Text(text = "Last item")
    }
}
```

可以使用`contentPadding`为内容添加内边距，使用`verticalArrangement`或`horizontalArrangement`，以`Arrangement.spacedBy()`为列表项之间添加间距

# 状态

在`Compose`中，数据的更新和传统命令式UI不同，是通过一种可观察类型对象，当一个可观察类型对象发生改变时，这个对象对应观察的部分会发生重组，从而自动更新UI

可观察类型`MutableState<T>`通常是通过`mutableStateOf()`函数创建的，这个对象的`value`发生变化时，对应UI也会跟着随之变化

```kt
//这里使用了kotlin的by关键字，是一种代理模式
//如果使用 = 的话，这个对象的类型会发生变化，需要count.value这样使用它的值
//var count = mutableStateOf(0)
var count by mutableStateOf(0)

@Composable
fun Demo(count: Int) {
    Column {
        Text(text = "count: ${count}")
        Button(onClick = { addCount() }) {
            Text(text = "add count")
        }
    }
}

fun addCount() {
    //++count.value
    ++count
}

@Preview
@Composable
fun Preview() {
    //当点击Button时，触发点击事件，更新可观察对象count，触发UI重组
    //Demo(count.value)
    Demo(count)
}
```

# 关于Context

在`Compose`中可以通过`LocalContext.current`获得当前`Context`

# 在传统View中使用Compose

可以在一个传统布局`xml`中插入一个`ComposeView`

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/hello_world"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Hello from XML layout" />

    <!-- 插入ComposeView -->
    <androidx.compose.ui.platform.ComposeView
        android:id="@+id/compose_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</LinearLayout>
```

然后在代码中设置这个`ComposeView`

```kt
findViewById<ComposeView>(R.id.compose_view).setContent {
	Text("Hello Compose!")
}
```

# 在Compose中使用传统View

可以使用`AndroidView`这个`composable`函数，这个函数接受一个`factory`参数，这个参数接受一个`Context`，用于构建传统`View`，要求返回一个继承自`View`的对象

```kt
@Composable
fun Demo() {
    Column {
        Text(text = "Compose Text")
        AndroidView(factory = { context ->
            //这里也可以使用LayoutInflater从xml中解析出一个View
            TextView(context).apply {
                text = "传统TextView"
            }
        })
    }
}
```

# 自定义UI

在`Compose`中，如果想要自定义一些简单的UI是很简单的，只需要写一个`Composable`函数就可以了，我们主要学习一下怎么自定义一些复杂的UI

我们先看一下怎么自定义一个布局，对应着传统UI中的`ViewGroup`，以一个简单的例子来说，我们自定义一个布局，让其中的子布局呈左上到右下依次排列：

```kt
@Composable
fun MyLayout(modifier: Modifier = Modifier, content: @Composable () -> Unit) {
    Layout(modifier = modifier, content = content) { measurables, constraints ->
        //测量每个子布局
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints)
        }

        //设置布局大小为最大可容纳大小
        layout(constraints.maxWidth, constraints.maxHeight) {
            var xPosition = 0
            var yPosition = 0

            //放置每个子View
            placeables.forEach { placeable ->
                placeable.placeRelative(x = xPosition, y = yPosition)

                //下一个子View的坐标为上一个子View的右下角
                xPosition += placeable.width
                yPosition += placeable.height
            }
        }
    }
}
```

我们再看一个使用`Canvas`自定义`View`的方式，这个更简单，就是画一条水平线：

```kt
@SuppressLint("ModifierParameter")
@Composable
fun HorizontalLine(modifier: Modifier = Modifier.fillMaxWidth()) {
    Canvas(modifier = Modifier
        .then(modifier), onDraw = {
        drawLine(color = Color.Black, Offset(0f, 0f), Offset(size.width, 0f), 2f)
    })
}
```

我们将两者一起用一下看看效果

```kt
@Preview(showBackground = true)
@Composable
fun Preview() {
    MyLayout {
        Text(text = "Text1")
        HorizontalLine(Modifier.width(50.dp))
        Text(text = "Text2")
    }
}
```

![效果图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a3f61a420a741e1b04f854a768cc501~tplv-k3u1fbpfcp-zoom-1.image)

其实`Compose`中的自定义UI的思路和传统自定义`View`是一样的，只不过需要熟悉`Compose`中的各种`Api`才能灵活运用它