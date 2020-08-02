---
title: Markdown语法
date: 2020-06-26 23:31:57
tags: Markdown
---

**Markdown**是一种轻量级标记语言，它以**纯文本**形式编写文档，并最终以**HTML格式**发布。

> 先推荐一款Markdown编辑器[Typora](https://typora.io/)，支持 Windows、OS X 和 Linux，支持即时渲染技术，所见即所得，界面简洁大方，非常好用。

**注：因为本站使用了主题，所以部分Markdown呈现的效果与原来不同**

---

# 标题

文字前加几个**#**即为几级标题

注：**#**与文字之间需要有**空格**

```markdown
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```

---

# 字体

在文字两侧加上特殊符号

```markdown
*倾斜*
**加粗**
***倾斜加粗***
~~删除线~~
```

*倾斜*
**加粗**
***倾斜加粗***
~~删除线~~

---

# 特殊符号

在特殊符号之前加上**\\**转义字符

```markdown
\\ 反斜杠
\* 星号
\_ 下划线
\{\} \[\] \(\) 括号
\+ 加号
```

\\ 反斜杠
\* 星号
\_ 下划线
\{\} \[\] \(\) 括号
\+ 加号

---

# 换行

连续两个以上的空格+回车，或两个以上的回车

---

# 分割线

在一行中用三个以上的**星号、减号、底线**

```markdown
***
***********
---
-----------
___
___________
```

可在中间插入空格，效果相同

---

# 引用

在引用文字前加 **>**

```markdown
> 这是一行引用文字
```

> 这是一行引用文字

引用可以嵌套

```
> 这是一行引用文字
>> 这是一行引用文字
>>> 这是一行引用文字
```

> 这是一行引用文字
> > 这是一行引用文字
> >
> > > 这是一行引用文字

---

# 列表

## 无序列表

在文字前加入**- + * 任意一个**符号

注：符号与文字之间需要有**空格**

```markdown
- 列表1
+ 列表2
* 列表3
```

* 列表1
* 列表2
* 列表3

## 有序列表

在文字前加入**数字和点**

```markdown
1. 列表1
2. 列表2
3. 列表3
```

1. 列表1
2. 列表2
3. 列表3

---

# 图片

格式 `![图片alt](图片地址 "图片title") Title可选`

```markdown
![Fate](/images/cover.jpg "Fate")
```

![Fate](/images/cover.jpg "Fate")

---

# 超链接

Markdown的超链接支持三种写法

## 行内式

格式 `[超链接名](超链接地址 "超链接title") 其中Title可选`

```markdown
[始终都是梦的Github-1](https://github.com/dreamgyf "始终都是梦的Github")
超链接名也可以嵌套图片
如 [![始终都是梦的Github-1](/images/avatar.jpeg)](https://github.com/dreamgyf)
```

[始终都是梦的Github 1](https://github.com/dreamgyf "始终都是梦的Github-1")

## 参考式

可以对一个链接进行多次引用

1. 使用 `[超链接名][超链接标记]`

2. 定义 `[超链接标记]:超链接地址 "超链接title" 其中Title可选,可定义在任意位置`

```markdown
[始终都是梦的Github-2][2]
[2]:https://github.com/dreamgyf
```

[始终都是梦的Github-2][2]

[2]:https://github.com/dreamgyf

## 自动链接

将链接直接用**<>**包裹起来

```markdown
<https://github.com/dreamgyf>
```

<https://github.com/dreamgyf>

---

# 脚注

1. 在需要添加注脚的文字后加上`[^脚注名]`

2. 在文本的任意位置(一般在最后)定义脚注 `[^脚注名]:脚注内容`

```markdown
示例文本[^1]
[^1]:我是示例文本的脚注
```

<font color=gray>由于本站主题原因不做演示</font>

---

# 代码

## 单行代码

将代码用特殊字符**\`**包裹(一般为Esc下面的键)

```markdown
`printf("hello world");`
```

`printf("hello world");`

## 代码块

使用两组三个特殊字符**\`**包裹代码块

可以在第一组**\```**后添加语言名

```markdown
​```c++
#include <iostream>

int main(void) {
	std::cout >> "hello world" >> endl;
	return 0;
}
​```  
```

```c++
#include <iostream>

int main(void) {
	std::cout >> "hello world" >> endl;
	return 0;
}
```

