---
title: 你真的了解Emoji吗？Emoji全貌大揭秘
date: 2024-08-10 15:13:23
tags: Emoji
categories: 其他
---

# 前言

随着科技发展，智能手机的普及，`Emoji`已经融入到了我们的生活中，但每天使用`Emoji`的你真的清楚它是什么，是由什么东西组成的，和普通的字符有什么区别吗？本文就从技术的角度带你揭秘`Emoji`的全貌。

# 起因

最近项目里有一个AI聊天机器人，你可以向他提问，他会以流式打印的形式一字一字的将回答呈现给用户。在这过程中我就发现，每当打印到`Emoji`的时候，总会先出现一个问号形状的乱码，然后才能显示出`Emoji`，有的`Emoji`更奇特，以👨‍👩‍👧‍👦为例，在打印过程中会依此显示👨👩👧👦四个`Emoji`，最后突然啪的一下，合成一整个👨‍👩‍👧‍👦，是不是很神奇？这个现象引起了我的好奇，于是我开始翻阅资料，揭开`Emoji`的神秘面纱。

# Emoji的起源及发展

`Emoji`来自日语词汇"絵文字"（假名为“えもじ”，读音即 emoji），绘指图画，文字指字符，最早由栗田穰崇（Shigetaka Kurita）创作，设计灵感源于天气预报图标、汉字、漫画和路标等，最初的`Emoji`有176个，都是12 x 12像素的图片。

1999年，日本通讯运营商DOCOMO公司发布了在当时具有跨时代意义的iMode手机，最早的`Emoji`便搭载于其中。

`Emoji`一经诞生，人们便发现这些形象的`Emoji`实在是太好用了，不仅方便，还能使聊天过程更加有趣，随即便立刻被日本各大科技公司注意到，日本的三大运营商开始把`Emoji`加入到自己的短信业务中，很快便横扫了全日本，但为了打击竞争对手，各大运营商都使用自己的`Emoji`标准。这导致了不同运营商的手机无法正常显示对方手机发的`Emoji`。

苹果是`Emoji`传遍全球的最大功臣。为了把`iPhone`打入日本市场，苹果决定在`iOS 2.2`中加入日本消费者的最爱emoji，为了迎合日本市场，他们在3个月的时间推出了400多个表情符号，极大地拓展了`Emoji`的表情数量，那时的`iOS Emoji`只在日本地区可用，但“好景不长”，北美的`iOS 2.2`用户发现了隐藏在系统中的`Emoji`，之后`Emoji`很快流行了起来，这种现象得到了其它科技公司的注意。

随着`Emoji`的流行，2010年，`Emoji` 首次被纳入`Unicode v6.0`字符集中。每个字符（表情）都被设定了统一且唯一的二进制码，从而保障了各平台手机都能使用`Emoji`，截止撰文期间，最新的`Unicode v15.1`字符集中已有3782个`Emoji`字符，而更新的`Unicode v16`版本预计于2024年9月发布release，届时会有更多的`Emoji`被支持。

# Unicode

既然`Emoji`被`Unicode`所收纳，那我们必先得去了解`Unicode`。

广义上的`Unicode`是一个标准，定义了`Unicode`字符集以及一系列的编码规则，是一种收录了世界上所有语言的文字和符号的全球标准。

那么`Unicode`是怎样收录如此庞大的字符内容呢？很简单，给每个字符指定一个编号就行了，在`Unicode`中被称为`码点`（`CodePoint`），它的表现形式为`U+`后面跟上一个十六进制数，比如`U+0041`表示大写字母`A`。

世界上有那么多字符，`Unicode`并不是一次性定义的，而是分区定义，每个区可以存放 65536 (`2^16`) 个字符，称为一个平面（Plane），目前`Unicode`从第0平面到第16平面总共有17个平面，其中第0平面被称为基本平面（BMP），它的码点范围从0一直到65535，写成十六进制也就是 `U+0000` - `U+FFFF` ，所有的常见字符都被放在这个平面，这是`Unicode`最先定义和公布的一个平面，而剩下的平面被称为辅助平面，码点范围从 `U+010000` 一直到 `U+10FFFF`

# UTF-16

`Unicode`字符集只规定了每个字符的码点，但这一个个码点应该被计算机传输识别呢？这就涉及到编码的概念了，目前`Unicode`实际应用使用的编码方式为`UCS-2`，也就是每个字符占用2个字节，`Unicode`还有一种4字节的编码方式`UCS-4`，但这里不做讨论。

使用`UCS-2`编码方式包含65536个字符空间（2个字节的可用空间即为`2^16`），对应着表示着`Unicode`字符集中的基本平面，那剩余的辅助平面又该如何表示呢？`UTF-16`应运而生。

`UTF-16`是`UCS-2`的超集，是一种变长编码，它的编码规则很简单：基本平面的字符占用2个字节，辅助平面的字符占用4个字节，也就是说`UTF-16`的编码长度要么是2个字节（`U+0000`-`U+FFFF`），要么是4个字节（`U+010000`-`U+10FFFF`）。

那么问题来了，采用`UTF-16`编码的时候，我们该怎么判断这个字符占用的是2个字节还是4个字节呢？这里有个巧妙的方式，在`Unicode`的基本平面中，从`U+D800`到`U+DFFF`是一个空段，即这些码点不对应任何字符，因此，这个空段可以用来映射辅助平面的字符。辅助平面的字符一共有`2^20`个（一个平面`2^16`个字符 * 16个平面(`2^4`)），因此表示这些字符至少需要20个二进制位。`UTF-16`将这20个二进制位分成两半，前10位映射在`U+D800`到`U+DBFF`（`UTF-16`的高半区，空间大小`2^10`），称为高位（H），后10位映射在`U+DC00`到`U+DFFF`（`UTF-16`的低半区，空间大小`2^10`），称为低位（L）。这意味着，一个辅助平面的字符，被拆成两个基本平面的字符表示。

因此，每当程序遇到2个字节时，便会去判断它的码元是否在`U+D800`到`U+DBFF`之间，如果在的话则可以假定它是一个4字节的字符，此时接着往后读2个字节，如果这2个字节的的码元在`U+DC00`到`U+DFFF`之间，将他们组合起来获得到实际字符；而如果不在的话则可以判定为是一个2字节的字符。

我们以`Java`中获取码点的方法`Character.codePointAt`为例来解读一下代码中如何获取一个字符的码点：

```java
public final
class Character implements java.io.Serializable, Comparable<Character> {

    // ...

    public static final char MIN_HIGH_SURROGATE = '\uD800';
    public static final char MAX_HIGH_SURROGATE = '\uDBFF';
    public static final char MIN_LOW_SURROGATE  = '\uDC00';
    public static final char MAX_LOW_SURROGATE  = '\uDFFF';

    public static final int MIN_SUPPLEMENTARY_CODE_POINT = 0x010000;

    // ...

    public static int codePointAt(CharSequence seq, int index) {
        char c1 = seq.charAt(index);
        // 码元在 U+D800 到 U+DBFF 之间，并且下一个char的index没到结尾
        if (isHighSurrogate(c1) && ++index < seq.length()) {
            char c2 = seq.charAt(index);
            // 下一个char的码元在 U+DC00 到 U+DFFF 之间
            if (isLowSurrogate(c2)) {
                // 组合起来获得完整字符的码点
                return toCodePoint(c1, c2);
            }
        }
        // 字符码点即是单个char的码元
        return c1;
    }

    // 码元是否在 U+D800 到 U+DBFF 之间
    public static boolean isHighSurrogate(char ch) {
        // Help VM constant-fold; MAX_HIGH_SURROGATE + 1 == MIN_LOW_SURROGATE
        return ch >= MIN_HIGH_SURROGATE && ch < (MAX_HIGH_SURROGATE + 1);
    }

    // 码元是否在 U+DC00 到 U+DFFF 之间
    public static boolean isLowSurrogate(char ch) {
        return ch >= MIN_LOW_SURROGATE && ch < (MAX_LOW_SURROGATE + 1);
    }

    /**
     * 计算规则：
     * 1. 高位上的码元减掉高半区的起始值 0xD800 ，然后左移10位
     * 2. 低位上的码元减掉低半区的起始值 0xDC00
     * 3. 将 1 和 2 的计算结果以及辅助平面的起始值 0x010000 相加，获取到完整的码点值
     */
    public static int toCodePoint(char high, char low) {
        // Optimized form of:
        // return ((high - MIN_HIGH_SURROGATE) << 10)
        //         + (low - MIN_LOW_SURROGATE)
        //         + MIN_SUPPLEMENTARY_CODE_POINT;
        return ((high << 10) + low) + (MIN_SUPPLEMENTARY_CODE_POINT
                                       - (MIN_HIGH_SURROGATE << 10)
                                       - MIN_LOW_SURROGATE);
    }
    
    // ...

}
```

# Emoji规则

了解了`Unicode`标准后，我们回过头来思考一下，是不是说`Emoji`在`Unicode`标准中也仅仅只是被当成普通的字符看待呢？当然并非如此，除了之前说的平面规则等，`Emoji`在`Unicode`中还有一套自己的规则，这些规则都可以在 [Unicode 技术标准 #51 Emoji](https://www.unicode.org/reports/tr51/) 中找到，官方的文档乍一看可能比较难理解，接下来就由我来给大家做一个解读。

首先，`Emoji`在大类上可以分成两种，一种是基本`Emoji`，一种是多字符组合而成的复合`Emoji`

## 基本Emoji

什么是基本`Emoji`呢？指的是直接在`Unicode`字符集里定义的一个`Emoji`字符，大多数基本`Emoji`字符都被划归到`U+1F300`-`U+1F6FF`和`U+1F900`-`U+1FAFF`这两个区域

![基本Emoji U+1F300-U+1F6FF](https://github.com/dreamgyf/ImageStorage/raw/master/你真的了解Emoji吗？Emoji全貌大揭秘_基本Emoji_U+1F300-U+1F6FF.png)

![基本Emoji U+1F900-U+1FAFF](https://github.com/dreamgyf/ImageStorage/raw/master/你真的了解Emoji吗？Emoji全貌大揭秘_基本Emoji_U+1F900-U+1FAFF.png)

具体都有哪些基本`Emoji`字符，我们可以在`Unicode`的官网文档 [emoji-data](https://www.unicode.org/Public/UCD/latest/ucd/emoji/emoji-data.txt) 中找到

## 复合Emoji

所谓的复合`Emoji`（我自己取的名字）指的是由多个字符组成的`Emoji`，它有着多种构造方式

在 [Unicode 技术标准 #51 Emoji](https://www.unicode.org/reports/tr51/) 中的`1.4.9`小节中，我们可以找到`Unicode`对`Emoji`定义的正则表达式，接下来我们就通过对这个正则表达式进行一步步的解析来了解`Emoji`的组成规则

```
\p{RI} \p{RI} 
| \p{Emoji} 
  ( \p{EMod} 
  | \x{FE0F} \x{20E3}? 
  | [\x{E0020}-\x{E007E}]+ \x{E007F}
  )?
  (\x{200D}
    ( \p{RI} \p{RI}
    | \p{Emoji}
      ( \p{EMod} 
      | \x{FE0F} \x{20E3}? 
      | [\x{E0020}-\x{E007E}]+ \x{E007F}
      )?
    )
  )*
```

### 旗帜

首先我们看第一行的匹配条件`\p{RI} \p{RI}`，这里的`\p{RI}`全称为`Regional Indicator`，翻译成中文就是区域指示符，根据这行正则我们可以了解到，两个区域指示符连接便可组成一个`Emoji`，那么这个区域指示符是什么呢？

通过 [维基百科](https://zh.wikipedia.org/wiki/%E5%8C%BA%E5%9F%9F%E6%8C%87%E7%A4%BA%E7%AC%A6) 我们可以得知，区域指示符指的是从`U+1F1E6`到`U+1F1FF`中的字符，位于`Unicode`第一辅助平面的带圈字母数字补充区块内。

正如区块描述所说，这些字符看起来就像是一个个英文字母，外面套了个方框，这里是完整的字符表：🇦 🇧 🇨 🇩 🇪 🇫 🇬 🇭 🇮 🇯 🇰 🇱 🇲 🇳 🇴 🇵 🇶 🇷 🇸 🇹 🇺 🇻 🇼 🇽 🇾 🇿，通过两两组合的方式，将它们拼成国家或地区的代号，我们就能得到该国家或地区的旗帜。

以中国🇨🇳举例，中国的代号为`CN`，那我们就将🇨和🇳两个字符拼接到一起，便能得到中国国旗🇨🇳

### 肤色修饰符

第二行的`\p{Emoji}`指的就是我们之前说过的基本`Emoji`，这个是构成除旗帜外的复合`Emoji`的基础条件，接着我们看正则的第三到第六行，这里用括号括起了一个条件，括号的尾部跟了一个问号，表示括号中的这个条件最多只可以出现一次（0次或1次），括号内的条件又是由三个子条件组成，用或号分割，满足任意一条条件则视为整个条件成立，我们首先看第一个子条件`\p{EMod}`。

全世界的人们都希望拥有反映更多人类多样性的`Emoji`，尤其是对于肤色。`Unicode v8.0`（2015年中）发行了五个为人类表情符号提供一系列肤色的符号修饰符符，具体的修饰符以及效果由下图所示：

![肤色修饰符](https://github.com/dreamgyf/ImageStorage/raw/master/你真的了解Emoji吗？Emoji全貌大揭秘_肤色修饰符.png)

我们以基本`Emoji`✋为例，它的码点是`U+270B`，在他后面加上`U+1F3FB`，这个`Emoji`就变成了✋🏻，同样的：

- ✋ + `U+1F3FC` = ✋🏼
- ✋ + `U+1F3FD` = ✋🏽
- ✋ + `U+1F3FE` = ✋🏾
- ✋ + `U+1F3FF` = ✋🏿

### 变体选择符

接着，我们再看第二个子条件`x{FE0F} \x{20E3}?`，这里的`x{FE0F}`指的是`变体选择符-16`（`Variation Selector-16`），那么首先，什么是变体选择符呢？

实际上，支持象形文字的字体最早可以追溯到1993年，我们可以看一下`Unicode`字符集的装饰符号区`U+2700`-`U+27FF`

![装饰符号区](https://github.com/dreamgyf/ImageStorage/raw/master/你真的了解Emoji吗？Emoji全貌大揭秘_装饰符号区.png)

那如果`Emoji`想要在这些象形文字的基础上做扩展，添加颜色，使其更加生动怎么办？没错，此时就需要使用到变体选择符了。

变体选择符（简称VS）是一个基本多文种平面的`Unicode`区段，包括16个变体选择符。这些选择器用于描述前一个字符的特点字形。目前 `Unicode`已定义数学符号、绘文字、八思巴字母及中日韩统一表意文字所对应的中日韩兼容表意文字。目前`Unicode`仅定义 VS1, VS2, VS3, VS15 及 VS16，VS15 和 VS16 分别用于标示某字符应该显示为普通文字或者是`Emoji`，这些字符被命名为`U+FE00`（VS1）至`U+FE0F`(VS16)。选择符仅应用于前一个字符。

以刚才我们在装饰符号区中看到的剪刀符号✂`U+2702`为例，在它的后面加上VS16 `U+FE0F`，这个字符就变成了`Emoji`✂️，是不是很神奇

### 键帽符

看完了变体选择符后，我们紧接着会疑惑，那这个条件后面的`\x{20E3}`又是啥呢？它被称为`COMBINING ENCLOSING KEYCAP`，它对前置的字符有一定的要求，只对数字、星号和井号生效，也就是说仅仅支持`*#0123456789`这12个字符。

它的规则是，当开头为这12个字符中的一个时，后面加上VS16 `U+FE0F`变成一个`Emoji`，然后在加上它`U+20E3`，这个字符就会变成一个键帽形状的字符：#️⃣ *️⃣ 0️⃣ 1️⃣ 2️⃣ 3️⃣ 4️⃣ 5️⃣ 6️⃣ 7️⃣ 8️⃣ 9️⃣

### 标签序列

然后是最后一个子条件`[\x{E0020}-\x{E007E}]+ \x{E007F}`，这是一个标签序列，它由一个基础黑旗符号，一系列标签字符以及一个标签终止符组成，首先以基础黑旗符号🏴`U+1F3F4`开头，然后中间是一系列的`U+E0020`到`U+E007E`之间的字符，最后以标签终止符`U+E007F`结尾，这样就组成了一个标签序列`Emoji`。

目前这种`Emoji`不太常见，仅仅只有英格兰、苏格兰和威尔士的旗帜使用标签序列：

- 🏴 + `U+E0067` + `U+E0062` + `U+E0065` + `U+E006E` + `U+E0067` + `U+E007F` = 🏴󠁧󠁢󠁥󠁮󠁧󠁿
- 🏴 + `U+E0067` + `U+E0062` + `U+E0073` + `U+E0063` + `U+E0074` + `U+E007F` = 🏴󠁧󠁢󠁳󠁣󠁴󠁿
- 🏴 + `U+E0067` + `U+E0062` + `U+E0077` + `U+E006C` + `U+E0073` + `U+E007F` = 🏴󠁧󠁢󠁷󠁬󠁳󠁿

### 零宽度连接符

这些子条件看完，我们再回归到正则表达式中来，可以观察到8到14行的条件和1到6行的条件其实是完全一样的，而在第7行出现了一个条件`\x{200D}`连接了这两个一样的条件，什么意思呢？

没错，结合本文的起因部分，我们很容易的就可以联想到，这个`\x{200D}`起到的就是连接作用，它被称为零宽度连接符（`ZERO-WIDTH JOINER`，简称`ZWJ`），通过上面正则表达式尾部的*号我们可以得知，通过这个连接符，可以连接多个`Emoji`合成新的`Emoji`，下面举几个有趣的例子：

- 👩`U+1F469` + `U+200D` + ✈️`U+2708 U+FE0F` = 👩‍✈️
- 👨`U+1F468` + `U+200D` + 💻`U+1F4BB` = 👨‍💻
- 🐻`U+1F43B` + `U+200D` + ❄️`U+2744 U+FE0F` = 🐻‍❄️
- 🏴`U+1F3F4` + `U+200D` + ☠️`U+2620 U+FE0F` = 🏴‍☠️
- 🏳️`U+1F3F3 U+FE0F` + `U+200D` + 🌈`U+1F308` = 🏳️‍🌈

以上是由两个`Emoji`组成一个新的`Emoji`的例子，而家庭以及人际关系相关的`Emoji`通常会由更多`Emoji`构成，就拿本文开头提到的例子👨‍👩‍👧‍👦，它的构成实际上是这样的：

👨`U+1F468` + `U+200D` + 👩`U+1F469` + 👧`U+1F467` + 👦`U+1F466` = 👨‍👩‍👧‍👦

这也就解释了在AI聊天机器人流式打印文字的时候，为什么会依此显示👨👩👧👦四个`Emoji`，最后突然合成一整个👨‍👩‍👧‍👦了

# Emoji字体

以上便是`Emoji`构成的所有规则了，但你有没有考虑过，一般字体都是黑白的矢量图形，为什么`Emoji`会显示成图片呢？

操作系统一般都会内置一种`Emoji`字体，`MacOS`/`iOS`内置的是`Apple Color Emoji`字体，`Windows`内置的是`Segoe UI Emoji`字体，`Android`内置的是`Noto Color Emoji`字体。这也是同一个`Emoji`再不同的设备上长得不一样的原因，除此之外，很多应用也会自带`Emoji`字体，比如`WhatsApp`、`Twitter`和`Facebook`

![Apple Color Emoji](https://github.com/dreamgyf/ImageStorage/raw/master/你真的了解Emoji吗？Emoji全貌大揭秘_Apple_Color_Emoji.png)

![不同平台下的Emoji样式](https://github.com/dreamgyf/ImageStorage/raw/master/你真的了解Emoji吗？Emoji全貌大揭秘_不同平台下的Emoji样式.png)

# 回归初心，如何解决流式打印问题

最后，让我们回到本文的出发点，了解了`Emoji`机制后，我们该如何解决AI聊天机器人的流式打印问题呢？其实很简单，根据字符串向后做一个预测就可以了，啥叫预测？就是往后匹配，看这个字符的结构当前是否符合`Emoji`规则，以及加上后面的字符后有没有可能组成一个完整的`Emoji`，这里具体的代码我就不放了，大家自行感悟吧😊

# 参考文献

- [Emoji 维基百科](https://zh.wikipedia.org/wiki/%E7%B9%AA%E6%96%87%E5%AD%97)
- [UTF-16 维基百科](https://zh.wikipedia.org/wiki/UTF-16)
- [区域指示符 维基百科](https://zh.wikipedia.org/wiki/%E5%8C%BA%E5%9F%9F%E6%8C%87%E7%A4%BA%E7%AC%A6)
- [总被大做文章的它从哪来？ Emoji起源和历史科普](https://tech.sina.com.cn/roll/2018-03-05/doc-ifyrztfz8047785.shtml)
- [emoji的诞生、发展背后，有哪些设计开发者的故事？](https://www.canva.cn/learn/emoji-design-and-development/)
- [Emoji](https://kawo.com/cn/%E7%A4%BE%E4%BA%A4%E5%AA%92%E4%BD%93%E6%9C%AF%E8%AF%AD%E8%A1%A8/emoji)
- [彻底弄懂Unicode编码](https://liyucang-git.github.io/2019/06/17/%E5%BD%BB%E5%BA%95%E5%BC%84%E6%87%82Unicode%E7%BC%96%E7%A0%81/)
- [Unicode？UTF-8？GBK？……聊聊字符集和字符编码格式](https://blog.hackerpie.com/posts/text-processing/character-sets-and-encoding-formats/)
- [Emoji的奥秘](https://taoshu.in/emoji.html)
- [Unicode 技术标准 #51 Emoji](https://www.unicode.org/reports/tr51/)

# 工具

- [Emoji Unicode Tables](https://apps.timwhitlock.info/emoji/tables/unicode)
- [Unicode character inspector](https://apps.timwhitlock.info/unicode/inspect)