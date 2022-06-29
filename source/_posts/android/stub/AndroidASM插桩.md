---
title: Android ASM插桩
date: 2022-03-01 15:32:00
tags: 插桩
categories: 
- [Android, 插桩]
---

# 简介

ASM插桩在网上其实已经有很多资料了，我之所以再写这篇文章呢，一是因为好久前学习的ASM，现在已经忘的差不多了，需要再回顾一下，二来是记录一下学习过程，以后如果再有细节记不清楚可以很方便的就能查到，三来再学习的过程中也踩了一些坑，收获了一些心得，这些也需要一个地方记录一下。

好了，废话就说到这里，接下来开始正文。

插桩技术指在保证原有程序逻辑完整性的基础上，在程序中插入探针，通过探针采集代码中的信息（方法本身、方法参数值、返回值等）在特定的位置插入代码段，从而收集程序运行时的动态上下文信息。

插桩技术大体可以分为两类：

- `APT`（Annotation Process Tools），在编译的时候，动态生成 `Java` 文件，之后编译器将生成的 `Java` 文件编译成 `class` 文件，像 `ButterKnife`、`Dagger` 就是通过 `APT` 的方式生成代码的。

  - 代表工具：`ButterKnife`

- `AOP`（Aspect Oriented Programming），生成 `class` 文件后，修改 `class` 文件的字节码，达到修改代码的目的。

  - 代表工具：听云

# 工具

我们这次选用`AOP`技术，我们看看有哪些工具可以帮助我们完成插桩工作：

- `AspectJ`，成熟稳定，使用者不需要对字节码文件有深入的理解，使用简单。但是其切入点相对固定，对于字节码文件的操作自由度以及开发的掌控度就大打折扣。并且，他会额外生成一些包装代码，对性能以及包大小有一定影响。

- `ASM`，可以修改现有的字节码文件，也可以动态生成字节码文件，完全从字节码去操作字节码的框架，更加灵活，功能更加强大，可以根据需求自定义修改、插入、删除，性能也十分出色，但是要对字节码文件有比较深入的了解，上手也更难。

我们使用`ASM`来完成插桩，在介绍`Android`字节码插桩之前，需要先了解一下`Java`字节码的概念和`Android`程序打包过程。

# 字节码

我们知道，`Java`程序是运行在`JVM`（`Java`虚拟机）上的，`Java`源代码首先会由编译器（`Java Compiler`）编译成包含了`Bytecode`（字节码）的`.class`文件，程序执行时，由类加载器(`class loader`)将该类的字节码加载到`JVM`中，`JVM`会解释执行相应的`Bytecode`。如下图所示：

![Java编译执行过程](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E5%AD%97%E8%8A%82%E7%A0%81.png)

为什么不直接彻底编译成机器码，而需要字节码这个中间产物呢？`Java`是一门跨平台的语言，为了实现一份源码，处处运行的效果，每个平台都有对应不同的`JVM`，它会将源码对应的指令翻译成对应平台能够理解的机器指令。那为什么不从源码直接解释执行呢，我个人认为这是因为直接从源码开始的编译，速度非常慢，出于性能的考虑，先将源码做一些预处理，处理为字节码，来减轻运行前的编译的性能开销。

在做插桩之前，我们先要记住一点：`Java` 字节码指令是基于堆栈操作的，因为大部分的`Java`虚拟机对字节码的执行是基于堆栈的（`Android`的`Dalvik`虚拟机是基于寄存器的，不过不影响我们的插桩，因为在我们对`java`字节码插完桩后，才会执行从`java`字节码转换到`dex`文件的过程）

# Android打包过程

![Android打包过程](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E6%89%93%E5%8C%85%E8%BF%87%E7%A8%8B.png)

# Android插桩过程

![Android插桩点](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E6%8F%92%E6%A1%A9%E8%BF%87%E7%A8%8B.png)

![Android插桩点](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E6%8F%92%E6%A1%A9%E8%BF%87%E7%A8%8B2.png)

# 实战

这次，我们模仿听云，做一个`Activity`生命周期执行时间检测的插件。

我们先梳理一下功能点：

1. 针对`Activity`类
2. 针对生命周期方法
3. 支持插件自定义配置

我们用`Java`代码把我们想要插入的逻辑写一遍：

```java
public class Test {

    //这里取这个名字是为了防止和代码本身的成员变量产生冲突
    private long _$_timeRecorder;

    public void onCreate(Bundle savedInstanceState) {
        //向实际代码前插入代码
	_$_timeRecorder = -System.currentTimeMillis();
        
	... //这里是实际代码
        
        //向实际代码后插入代码
	_$_timeRecorder += System.currentTimeMillis();
	System.err.println("Time spent: " + _$_timeRecorder + "ms, when " + className + ".onCreate");
	StackTraceElement[] stackTraceElements = Thread.currentThread().getStackTrace();
	for (StackTraceElement element : stackTraceElements) {
		System.err.println(element.getClassName() + "." + element.getMethodName() + ":" + element.getLineNumber());
	}
    }
}
```

接下来正式开始编写插件

## 新建插件工程

由于`Android Studio`没有新建`gradle`脚本的选项，我们先新建一个`Empty Activity Project`，在此基础上进行改造。

1.  新建`module`
2.  更改`module`的`build.gradle`文件
3.  新建`groovy`源代码目录
4.  新建`groovy`类实现`Plugin<Project>`接口
5.  新建`resource/META_INF/xxx.properites`文件（xxx为插件的id名）
6.  在`properites`文件中声明插件的实现类

## 为插件提供可配置的功能

1.  新建一个实体类用来保存配置信息

```java
public class AsmConfigModel {
	/**
	 * 以此参数为开头的类（全限定类名）才插桩
	 * 如果不配此参数则代表所有类都可插桩
	 */
	public List<String> startWithPatterns;
	/**
	 * 排除列表（全限定类名）
	 */
	public List<String> excludes;
	/**
	 * 排除列表（全限定类名）
	 * 以文件形式
	 */
	public File excludesByFile;
}
```

2.  在插件`apply`的时候创建这个配置类，以提供给使用者配置

```groovy
@Override
void apply(Project project) {
    println 'apply AsmPlugin'
    mConfig = project.extensions.create("asmConfig", AsmConfigModel.class)
}
```

3.  在使用该插件的`module`下的`build.gradle`文件中配置

```gradle
asmConfig {
    startWithPatterns = ['com.shanbay']
    excludesByFile = new File(projectDir, "asm-excludes.txt")
}
```

4.  新建asm-excludes.txt文件，配置exclude信息

```
com/xxx/xxx/BaseActivity
```

这里是举个例子，在工程中很有可能有的`Activity`继承自一些基类`Activity`，对这些类插桩就重复了

## 使用Transform Api

根据[官网](http://tools.android.com/tech-docs/new-build-system/transform-api)介绍，`Transform Api`允许第三方 `Plugin` 在打包 `dex` 文件之前的编译过程中操作`.class` 文件，下图是`Transform Api`的工作流程

![Transform Api工作流程](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_Transform%E8%BF%87%E7%A8%8B.png)

可以看到，一次`App`的编译打包可能会经历多次`Transform`，`Transform`将输入进行处理，然后写入到指定的目录下作为下一个 `Transform` 的输入源。

使用插桩工具，我们需要借助于`Transform Api`实现

1. 首先，我们需要让我们的插件继承自`Transform`
2. 然后，我们要在插件`apply`时注册`Transform`

```groovy
@Override
void apply(Project project) {
    println 'apply AsmPlugin'
    def android = project.extensions.getByType(AppExtension.class)
    android.registerTransform(this)
    mConfig = project.extensions.create("asmConfig", AsmConfigModel.class)
}
```
3. 最后，需要实现`Transform`类中的抽象方法

![Transform抽象方法](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_Transform%E7%B1%BB.png)

- `getName` 这个方法是指定这个`Transform`的名称

```groovy
@Override
String getName() {
    return 'AsmPlugin'
}
```

- `getInputTypes` 这个方法是指定输入类型

![Transform输入类型](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_getInputTypes.png)

![Transform输入类型](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_getInputTypes2.png)

这里，我们选用`TransformManager.CONTENT_CLASS`就可以了

- `getScopes` 这个方法是指定插桩的作用域

![Transform作用域](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_getScopes.png)

![Transform作用域](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_getScopes2.png)

这里我们选择`TransformManager.SCOPE_FULL_PROJECT`，代表插桩范围包括此工程和它依赖的所有包

- `isIncremental` 这个方法代表是否开启增量编译

如果开启的话可以减少编译时间，但需要增加额外的判断条件，所以这里就先不开启了

- `transform` 这个方法是核心方法，我们要对输入内容进行处理然后输出

`transform()`方法的参数 `TransformInvocation` 是一个接口，提供了一些关于输入输出的一些基本信息。下图是`transform`中我们需要走的流程

![Transform流程](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_transform%E6%B5%81%E7%A8%8B.png)

这里以`directoryInputs`举例，`directoryInputs`就是本地源码编译后产生的`class`文件

```groovy
private void handleDirectory(DirectoryInput input, TransformOutputProvider outputProvider) {
    File file = input.file

    if (file.isDirectory()) {
        //递归遍历该文件夹下面所有的子文件夹以及子文件
        file.eachFileRecurse { subFile ->
            def fileName = subFile.name
            //初步判断这个文件（或文件夹）是否可插桩
            if (fileName.endsWith(".class") && !fileName.startsWith("R$")
                    && "R.class" != fileName && "BuildConfig.class" != fileName) {
                //ClassReader: 字节码的读取与分析引擎
                ClassReader classReader = new ClassReader(subFile.bytes)
                //ClassWriter: 它实现了ClassVisitor接口，用于拼接字节码
                //COMPUTE_MAXS: 自动计算栈的最大值以及本地变量的最大数量
                //COMPUTE_FRAMES: 包含COMPUTE_MAXS，且会自动计算方法的栈桢
                ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
                //ClassVisitor: 定义在读取Class字节码时会触发的事件，如类头解析完成、注解解析、字段解析、方法解析等
                ClassVisitor cv = new AsmClassVisitor(api, classWriter, mConfig)
                //使给定的ClassVisitor访问传递给此构造函数的jvm类文件结构
                //EXPAND_FRAMES: 展开栈帧的标志位
                classReader.accept(cv, ClassReader.EXPAND_FRAMES)
                FileOutputStream fos = new FileOutputStream(
                        subFile.parentFile.absolutePath + File.separator + fileName)
                fos.write(classWriter.toByteArray())
                fos.close()
            }
        }
    }

    def dest = outputProvider.getContentLocation(
            input.name,
            input.contentTypes,
            input.scopes,
            Format.DIRECTORY
    )
    FileUtils.copyDirectoryToDirectory(file, dest)
}
```

可以用以下流程图大概描述一下一个`class`文件的修改过程

![class文件修改流程](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_class%E4%BF%AE%E6%94%B9%E8%BF%87%E7%A8%8B.png)

## 自定义ClassVisitor

我们开始继承`ClassVisitor`来实现我们对类的修改

### 读取配置

![读取配置](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E8%AF%BB%E5%8F%96%E9%85%8D%E7%BD%AE.png)

### 访问类

![访问类方法](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E8%AE%BF%E9%97%AE%E7%B1%BB.png)

通过这个方法我们可以获得这个类的访问控制，全限定类名，父类名，实现的接口名等信息

这里，我们通过全限定类名和读取出的配置做比对，进一步验证是否需要对此类进行插桩

![验证类是否可插桩](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E7%AD%9B%E9%80%89%E7%B1%BB.png)

![验证类是否可插桩](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E7%AD%9B%E9%80%89%E7%B1%BB2.png)

### 访问类内方法

![访问类内方法](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E8%AE%BF%E9%97%AE%E7%B1%BB%E5%86%85%E6%96%B9%E6%B3%95.png)

通过这个方法我们可以获得这个类的所有方法的名称和描述符，我们通过它们来判断该方法是否需要插桩

![判断方法是否需要插桩](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E7%AD%9B%E9%80%89%E6%96%B9%E6%B3%95.png)

如果有需要插桩的方法，就将`mNeedStubClass`标志位置为true，这个标识是为了我们后续判断是否要在该类中插入成员变量，然后使用我们自定义的`MethodVisitor`替换原始的`MethodVisitor`。

### 插入成员变量

![插入成员变量](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%20ASM%E6%8F%92%E6%A1%A9_%E6%8F%92%E5%85%A5%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F.png)

在最后，如果有需要插桩的方法，我们需要将`private long _$_timeRecorder`这个成员变量插入到类中去

## 自定义MethodVisitor

之前说了，`Java` 字节码指令是基于栈操作的，基本上任何操作都会改变栈状态

### 在方法执行之前插入代码

```java
/**
* 以下代码会以栈的形式注释出来，以左边为栈顶，右边为栈底
* 示例：[栈顶 <------------------> 栈底]
* [this, StringBuilder, System.out]
* 此时，this为栈顶，System.out为栈底
*/
@Override
public void visitCode() {

    /*
        假设此时栈为空
    */

    //aload_0: 将this压入栈顶
    mv.visitVarInsn(Opcodes.ALOAD, 0);

    /*
        此时栈内容:
        [this]
    */

    //invokestatic: 调用静态方法System.currentTimeMillis()，返回值为基础类型long
    //第二个参数代表类的全限定名，第三个参数代表方法名，第四个参数代表函数签名，()J的意思是不接受参数，返回值为J (J在字节码里代表基础类型long)
    mv.visitMethodInsn(Opcodes.INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);

    /*
        此时栈内容:
        [System.currentTimeMillis()的结果值, this]
    */

    //lneg: 将栈顶的long类型取负并将结果压入栈顶
    mv.visitInsn(Opcodes.LNEG);

    /*
        此时栈内容:
        [System.currentTimeMillis()的结果值取负, this]
    */

    //putfield: 为该类的此实例变量赋值
    //以(栈顶 - 1)为执行对象，为其赋值为栈顶值 (this._$_timeRecorder = -System.currentTimeMillis())
    mv.visitFieldInsn(Opcodes.PUTFIELD, mClassName, TIMER_NAME, "J");
    super.visitCode();
}
```

### 在方法return之前插入代码

```java
/**
* 以下代码会以栈的形式注释出来，以左边为栈顶，右边为栈底
* 示例：[栈顶 <------------------> 栈底]
* [this, StringBuilder, System.out]
* this为栈顶，System.out为栈底
*/
@Override
public void visitInsn(int opcode) {
    if (opcode == Opcodes.RETURN) {
        Label labelEnd = new Label();

        /*
            假设此时栈为空
        */

        //aload_0: 将this压入栈顶
        mv.visitVarInsn(Opcodes.ALOAD, 0);
        //dup: 将栈顶的值复制一份压入栈顶
        mv.visitInsn(Opcodes.DUP);

        /*
            此时栈内容:
            [this, this]
        */

        //以当前栈顶的值为主体，获取当前类的成员变量_$_timeRecorder，类型为long
        //相当于this._$_timeRecorder
        mv.visitFieldInsn(Opcodes.GETFIELD, mClassName, TIMER_NAME, "J");

        /*
            此时栈内容:
            [this._$_timeRecorder, this]
        */

        //执行System.currentTimeMillis()，并将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKESTATIC, "java/lang/System", "currentTimeMillis", "()J", false);

        /*
            此时栈内容:
            [System.currentTimeMillis()执行后的结果值, this._$_timeRecorder, this]
        */

        //将栈顶两long值相加，并将结果压入栈顶
        //即this._$_timeRecorder + System.currentTimeMillis
        mv.visitInsn(Opcodes.LADD);

        /*
            此时栈内容:
            [System.currentTimeMillis() + this._$_timeRecorder, this]
        */

        //将栈顶的值存入(栈顶 - 1)._$_timeRecorder中
        //即this._$_timeRecorder = this._$_timeRecorder + System.currentTimeMillis
        mv.visitFieldInsn(Opcodes.PUTFIELD, mClassName, TIMER_NAME, "J");

        /*
            此时栈为空
        */

        //L: 对象类型，以分号结尾，如Ljava/lang/Object;
        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");

        /*
            此时栈内容:
            [System.out]
        */

        //构建字符串
        //创建一个StringBuilder对象，此时还并没有执行构造方法
        mv.visitTypeInsn(Opcodes.NEW, "java/lang/StringBuilder");
        //因为执行构造函数会将栈顶的StringBuilder对象弹出，为了后续能继续使用这个对象，所以这里需要先复制一份
        mv.visitInsn(Opcodes.DUP);

        /*
            此时栈内容:
            [StringBuilder, StringBuilder, System.out]
        */

        //以栈顶的StringBuilder调用构造方法
        mv.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
        */

        //将常量压入栈顶
        mv.visitLdcInsn("Time spent: ");

        /*
            此时栈内容:
            ["Time spent: ", StringBuilder, System.out]
        */

        //以栈顶的值为参数，(栈顶 - 1)的引用为主体执行StringBuilder.append()方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
        */

        //将this压入栈顶
        mv.visitVarInsn(Opcodes.ALOAD, 0);

        /*
            此时栈内容:
            [this, StringBuilder, System.out]
        */

        //以当前栈顶的值为主体，获取当前类的成员变量_$_timeRecorder，类型为long
        //相当于this._$_timeRecorder
        mv.visitFieldInsn(Opcodes.GETFIELD, mClassName, TIMER_NAME, "J");

        /*
            此时栈内容:
            [this._$_timeRecorder, StringBuilder, System.out]
        */

        //以栈顶的值为参数，(栈顶 - 1)的引用为主体执行StringBuilder.append()方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(J)Ljava/lang/StringBuilder;", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
        */

        //将常量压入栈顶
        mv.visitLdcInsn("ms, when " + mFormatClassName + "." + mMethodName + ":" + mMethodDescriptor);

        /*
            此时栈内容:
            [字符串常量, StringBuilder, System.out]
        */

        //以栈顶的值为参数，(栈顶 - 1)的引用为主体执行StringBuilder.append()方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
        */

        //以栈顶的值为主体，执行StringBuilder.toString()方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);

        /*
            此时栈内容:
            [String, System.out]
        */

        //以栈顶的值为参数，(栈顶 - 1)的引用为主体执行PrintStream.println()方法
        //相当于System.out.println(String)
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

        /*
            此时栈为空
        */

        //执行Thread.currentThread()，并将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKESTATIC, "java/lang/Thread", "currentThread", "()Ljava/lang/Thread;", false);

        /*
            此时栈内容:
            [Thread.currentThread()执行的结果]
        */

        //以栈顶的值为主体，执行getStackTrace()方法，将返回值压入栈顶
        //相当于Thread.currentThread().getStackTrace()
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/Thread", "getStackTrace", "()[Ljava/lang/StackTraceElement;", false);

        /*
            此时栈内容:
            [StackTraceElement数组]
        */

        //astore: 将一个引用类型对象保存到局部变量表index为2的位置（index1: this, index2: onCreate方法的参数）
        //使用一个临时变量保存StackTraceElement数组
        mv.visitVarInsn(Opcodes.ASTORE, 2);
        //将局部变量表index2处的引用对象压入栈顶
        mv.visitVarInsn(Opcodes.ALOAD, 2);

        /*
            此时栈内容:
            [StackTraceElement数组]
            此时局部变量表中:
            [ 0        1             2           ]
            [this | Bundle | StackTraceElement数组]
        */

        //StackTraceElement数组备份
        mv.visitVarInsn(Opcodes.ASTORE, 3);
        mv.visitVarInsn(Opcodes.ALOAD, 3);

        /*
            此时栈内容:
            [StackTraceElement数组]
            此时局部变量表中:
            [ 0        1             2                       3           ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组]
        */

        //获得栈顶位置数组的长度
        mv.visitInsn(Opcodes.ARRAYLENGTH);

        /*
            此时栈内容:
            [StackTraceElement数组长度]
            此时局部变量表中:
            [ 0        1             2                       3           ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组]
        */

        //将数组length保存至局部变量表index4的位置
        mv.visitVarInsn(Opcodes.ISTORE, 4);

        /*
            此时栈为空
            此时局部变量表中:
            [ 0        1             2                       3                 4   ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度]
        */

        //将int常量0压入栈顶
        mv.visitInsn(Opcodes.ICONST_0);
        //将栈顶的0取出保存（用作循环下标index）
        mv.visitVarInsn(Opcodes.ISTORE, 5);

        /*
            此时栈为空
            此时局部变量表中:
            [ 0        1             2                       3                 4          5    ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index]
        */

        //循环开始处
        //插入一个label用来做后续循环跳转的标志
        Label labelLoop = new Label();
        mv.visitLabel(labelLoop);
        //将循环标志位的值压入栈顶
        mv.visitVarInsn(Opcodes.ILOAD, 5);
        //将数组长度值压入栈顶
        mv.visitVarInsn(Opcodes.ILOAD, 4);

        /*
            此时栈内容:
            [循环标志位, 数组长度]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5    ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index]
        */

        //if_icmpge: 比较栈顶两int型数值大小, 当结果大于等于0时跳转
        mv.visitJumpInsn(Opcodes.IF_ICMPGE, labelEnd);

        /*
            此时栈为空
            此时局部变量表中:
            [ 0        1             2                       3                 4          5    ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index]
        */

        //将StackTraceElement数组压入栈顶
        mv.visitVarInsn(Opcodes.ALOAD, 3);
        //将循环index的值压入栈顶
        mv.visitVarInsn(Opcodes.ILOAD, 5);

        /*
            此时栈内容:
            [循环index, StackTraceElement数组]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5    ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index]
        */

        //将引用类型数组指定索引的值推送至栈顶（var3[var5]）
        mv.visitInsn(Opcodes.AALOAD);

        /*
            此时栈内容:
            [StackTraceElement数组中的某个值(以循环index作为下标)]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5    ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index]
        */

        //将该索引下的值保存
        mv.visitVarInsn(Opcodes.ASTORE, 6);

        /*
            此时栈为空
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //将System.out入栈
        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");

        /*
            此时栈内容:
            [System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //new StringBuilder()
        mv.visitTypeInsn(Opcodes.NEW, "java/lang/StringBuilder");
        mv.visitInsn(Opcodes.DUP);
        mv.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //取出StackTraceElement数组中的某个值(以循环index作为下标)
        mv.visitVarInsn(Opcodes.ALOAD, 6);

        /*
            此时栈内容:
            [StackTraceElement数组中的某个值(以循环index作为下标), StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //使用栈顶对象，执行getClassName方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StackTraceElement", "getClassName", "()Ljava/lang/String;", false);

        /*
            此时栈内容:
            [ClassName, StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //以ClassName作为参数，执行(栈顶 - 1)对象的append方法，将返回值压入栈顶
        //即StringBuilder.append(ClassName)
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //将常量入栈
        mv.visitLdcInsn(".");
        //以常量作为参数，执行(栈顶 - 1)对象的append方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //将StackTraceElement数组中的某个值(以循环index作为下标)入栈
        mv.visitVarInsn(Opcodes.ALOAD, 6);
        //调用它的getMethodName方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StackTraceElement", "getMethodName", "()Ljava/lang/String;", false);

        /*
            此时栈内容:
            [MethodName, StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //以MethodName作为参数，执行(栈顶 - 1)对象的append方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //将常量入栈
        mv.visitLdcInsn(":");
        //以常量作为参数，执行(栈顶 - 1)对象的append方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //将StackTraceElement数组中的某个值(以循环index作为下标)入栈
        mv.visitVarInsn(Opcodes.ALOAD, 6);
        //调用它的getLineNumber方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StackTraceElement", "getLineNumber", "()I", false);

        /*
            此时栈内容:
            [LineNumber, StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //以LineNumber作为参数，执行(栈顶 - 1)对象的append方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(I)Ljava/lang/StringBuilder;", false);

        /*
            此时栈内容:
            [StringBuilder, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //调用栈顶对象的toString方法，将返回值压入栈顶
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);

        /*
            此时栈内容:
            [String, System.out]
            此时局部变量表中:
            [ 0        1             2                       3                 4          5                             6                         ]
            [this | Bundle | StackTraceElement数组 | StackTraceElement数组 | 数组长度 | 循环index | StackTraceElement数组中的某个值(以循环index作为下标)]
        */

        //以String作为参数，执行(栈顶 - 1)对象System.out的println方法
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

        //iinc: 将指定int型变量增加指定值(index++)
        mv.visitIincInsn(5, 1);
        //跳转到labelLoop插入的位置
        mv.visitJumpInsn(Opcodes.GOTO, labelLoop);

        //插入结束Label，作为循环终止的跳转标志
        mv.visitLabel(labelEnd);
    }

    super.visitInsn(opcode);
}
```

这样我们的方法插桩工作就完成了，接下来我们运行一下看看

## 运行

先`clean build`，再`build`，查看控制台信息，`build`完成后查看`class`文件

运行`App`，查看`Logcat`信息，可以看到打印出来了我们想要的信息。

# 结语

这样我们就通过插桩的方式，实现了一个简单的无任何代码侵入的性能检测工具

通过这一次实践，我对`java`的编译运行字节码，`Android`的打包流程有了更深的理解

完整项目地址：<https://github.com/dreamgyf/AsmPluginDemo>