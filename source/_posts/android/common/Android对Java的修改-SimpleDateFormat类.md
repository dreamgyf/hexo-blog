---
title: Android对Java的修改-SimpleDateFormat类
date: 2021-03-01 17:06:00
tags: SimpleDateFormat
categories: Android
---

# 简介

Android会对部分OpenJDK中的代码进行一些修改，本篇记录一下因为这些修改而踩过的一些坑。

# 问题描述

一个在线上运行良好的Date工具类在写单元测试时一直报ParseException，代码如下：

```java
public static String utc2Local(String utcTime) {
    String utcTimePatten = "yyyy-MM-dd'T'HH:mm:ssZZZZZ";
    String localTimePatten = "yyyy.MM.dd";
    SimpleDateFormat utcFormater = new SimpleDateFormat(utcTimePatten);
    utcFormater.setTimeZone(TimeZone.getTimeZone("UTC"));//时区定义并进行时间获取
    Date gpsUTCDate = null;
    try {
        gpsUTCDate = utcFormater.parse(formatTimeStr(utcTime));
    } catch (Exception e) {
        e.printStackTrace();
        return utcTime;
    }
    SimpleDateFormat localFormater = new SimpleDateFormat(localTimePatten);
    localFormater.setTimeZone(TimeZone.getDefault());
    String localTime = localFormater.format(gpsUTCDate.getTime());
    return localTime;
}
```

这里传入的参数utcTime为"2020-01-01 08:00:00+08:00"

这段代码在Android环境下运行良好，但在单元测试下一直报错

# 原因

Android对OpenJDK中的`SimpleDateFormat`进行了修改，具体在`subParseNumericZone`方法中：

![](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E5%AF%B9Java%E7%9A%84%E4%BF%AE%E6%94%B9-SimpleDateFormat%E7%B1%BB.png)

可以看到，OpenJDK原本是不支持带冒号的写法的，而在Android中修改了`subParseNumericZone`方法，使其可以解析带冒号的写法。

# 解决

解决方法也很简单，在测试时直接修改入参，去掉入参中的冒号就好了：

```java
try (MockedStatic<DateUtils> mockedDateUtils = Mockito.mockStatic(DateUtils.class, new CallsRealMethods())) {
    mockedDateUtils.when(() -> {
       	DateUtils.utc2Local(argThat((argument) -> {
            int index = argument.length() - 3;
            return argument.charAt(index) == ':';
        ));
    }).then((invocation) -> {
        String utcTime = invocation.getArgument(0, String.class);
        String fixedDate = formatDate;
        int index = formatDate.length() - 3;
        if (formatDate.charAt(index) == ':') {
            fixedDate = formatDate.substring(0, index) + formatDate.substring(index + 1);
        }
        return DateUtils.utc2Local(fixedDate);
    });
}
```