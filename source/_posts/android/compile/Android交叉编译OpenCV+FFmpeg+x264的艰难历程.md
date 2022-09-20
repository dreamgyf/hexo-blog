---
title: Android交叉编译OpenCV+FFmpeg+x264的艰难历程
date: 2022-09-14 14:55:13
tags: 
- Android交叉编译
- NDK
- OpenCV
- FFmpeg
- x264
categories: 
- [Android, 交叉编译]
- [Android, NDK]
- 编译
---

# 前言

如果你没有兴趣看完本文，只想获得可编译的代码或编译后的产物，可以直接点击下面的链接，跟随步骤编译代码或直接下载我编译好的产物

**注：编译顺序要按照 x264 -> FFmpeg -> OpenCV 这样来**

[x264](https://github.com/dreamgyf/x264/releases/tag/v0.164_compilable)

[FFmpeg](https://github.com/dreamgyf/FFmpeg/releases/tag/v5.0_compilable)

[OpenCV](https://github.com/dreamgyf/opencv/releases/tag/v4.6.0_compilable)

# 起因

最近在做一个视频生成的app，使用`OpenCV`库实现，用的是C语言，一开始我是在`mac_x86`上书写代码，`fourcc`视频编码器选择的是`mp4v`，视频输出一切正常，然后当我将代码移植到`Android`上时发现，从`OpenCV`官网下载的`so`库它不支持编码`mp4v`格式，只能编码成`mjpg`格式，后缀名为`avi`，尴尬的是`Android`原生又不支持播放这种格式的视频，所以要想办法让`OpenCV`支持编码`mp4v`或`h264`等格式

我在网上搜索了一下为什么`OpenCV`默认不支持`h264`格式，得知`OpenCV`默认使用`FFmpeg`做视频处理，`FFmpeg`使用的是`LGPL`协议，而`x264`使用的是`GPL`协议，`GPL`协议具有传染性，如果代码中使用了`GPL`协议的软件，则要求你的代码也必须开源。我猜测是因为这个原因，`FFmpeg`默认不使用`GPL`协议的软件，避免产生一些不必要的问题和纠纷，如果想要使用`GPL`协议的软件，则需要在编译的时候加上`--enable-gpl`选项

基于此上原因，我开启了我艰难的编译之路

# 声明

本篇文章只针对`Linux`系统编译，其他系统不保证可以编译通过

本篇文章使用的`NDK`版本为`21.4.7075529`，不同的版本可能会有些差别，需要自行调整

本人对`c/c++`编译这块并不是很了解，很多东西也是边学习边尝试的，如果有什么错误的话也恳请大佬们指正，谢谢

# 准备

准备一台`Linux`系统的电脑或使用虚拟机，安装一些最基本的编译工具（`make`、`cmake`等），我使用的是`Ubuntu`系统，强烈建议在安装的时候选择完整安装，这样这些编译工具应该都会跟随系统自动安装好

`Android`交叉编译肯定是需要`NDK`的，我使用的是`21.4.7075529`版本，`r19`以上版本的NDK都是直接自带了工具链，而`r19`之前的版本则需要先生成工具链，具体可以参考[独立工具链（已弃用）](https://developer.android.com/ndk/guides/standalone_toolchain?hl=zh-cn)这篇文档

# x264

既然需要依赖`x264`，那我们肯定是先要编译`x264`库，各位可以`clone`我准备好的`tag`

```shell
git clone -b v0.164_compilable https://github.com/dreamgyf/x264.git
```

这个版本是从原`x264`镜像仓库的`stable`分支切出的，版本为`0.164`。想知道`x264`版本的话，可以运行其目录下的`version.sh`脚本，它会输出三串数字，前面的`164`是在`x264.h`中定义的`X264_BUILD`，第二个`3095+4`表示`master`分支的提交数 + `master`分支到HEAD的提交数，最后的一串数字表示当前分支最新的`commit id`

在构建编译脚本之前，我们先要看看这个库提供了哪些编译选项，我们可以看到在`x264`根目录下有一个`configure`文件，这是一个脚本文件，大多数库都提供了这个脚本，用来负责生成`Makefile`，准备好构建环境，我们可以通过下面这个命令获取帮助文件

```shell
./configure --help > help.txt
```

可以看到，里面提供了一些编译选项及其描述，我们可以根据这些选项和描述构建编译脚本

先看一下我写好的脚本吧

```shell
# Linux 交叉编译 Android 库脚本
if [[ -z $ANDROID_NDK ]]; then
    echo 'Error: Can not find ANDROID_NDK path.'
    exit 1
fi

echo "ANDROID_NDK path: ${ANDROID_NDK}"

OUTPUT_DIR="_output_"

mkdir ${OUTPUT_DIR}
cd ${OUTPUT_DIR}

OUTPUT_PATH=`pwd`

API=21
TOOLCHAIN=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64

function build {
    ABI=$1

    if [[ $ABI == "armeabi-v7a" ]]; then
        ARCH="arm"
        TRIPLE="armv7a-linux-androideabi"
    elif [[ $ABI == "arm64-v8a" ]]; then
        ARCH="arm64"
        TRIPLE="aarch64-linux-android"
    elif [[ $ABI == "x86" ]]; then
        ARCH="x86"
        TRIPLE="i686-linux-android"
    elif [[ $ABI == "x86-64" ]]; then
        ARCH="x86_64"
        TRIPLE="x86_64-linux-android"
    else
        echo "Unsupported ABI ${ABI}!"
        exit 1
    fi

    echo "Build ABI ${ABI}..."

    rm -rf ${ABI}
    mkdir ${ABI} && cd ${ABI}

    PREFIX=${OUTPUT_PATH}/product/$ABI

    export CC=$TOOLCHAIN/bin/${TRIPLE}${API}-clang
    export CFLAGS="-g -DANDROID -fdata-sections -ffunction-sections -funwind-tables -fstack-protector-strong -no-canonical-prefixes -D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security  -O0 -DNDEBUG  -fPIC --gcc-toolchain=$TOOLCHAIN --target=${TRIPLE}${API}"

    ../../configure \
        --host=${TRIPLE} \
        --prefix=$PREFIX \
        --enable-static \
        --enable-shared \
        --enable-pic \
        --disable-lavf \
        --sysroot=$TOOLCHAIN/sysroot

    make clean && make -j`nproc` && make install

    cd ..
}

echo "Select arch:"
select arch in "armeabi-v7a" "arm64-v8a" "x86" "x86-64"
do
    build $arch
    break
done
```

这也是我其他库编译脚本的基本结构，首先需要`ANDROID_NDK`环境变量用来确定`NDK`的位置

`OUTPUT_DIR`为编译的输出路径，我这里命名为`_output_`，防止和源码本身的目录重名

`API`为最低支持的`Android API`版本，我这里写的`21`，也就是`Android 5.0`

`TOOLCHAIN`为交叉编译工具链的路径，对于`r19`之前的`NDK`，需要将其改为你生成出来的工具链的路径，`r19`之后不需要改动

我这里定义了一个`build`函数，通过输入的`ABI`编译出对应架构的产物。`ABI`总共有四种：`armeabi-v7a`，`arm64-v8a`，`x86`，`x86-64`，这个决定你的`App`能在哪些平台架构上运行

这里，我通过不同的`ABI`定义了不同的`TRIPLE`变量，这是遵循了`NDK`工具链的命名规则，可以在 [将 NDK 与其他构建系统配合使用](https://developer.android.com/ndk/guides/other_build_systems?hl=zh-cn) 这篇文档中找到

![TRIPLE](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91OpenCV%2BFFmpeg%2Bx264%E7%9A%84%E8%89%B0%E9%9A%BE%E5%8E%86%E7%A8%8B_TRIPLE.png)

在`$TOOLCHAIN/bin`目录下，我们也能发现这种命名方式

![TRIPLE](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91OpenCV%2BFFmpeg%2Bx264%E7%9A%84%E8%89%B0%E9%9A%BE%E5%8E%86%E7%A8%8B_TRIPLE2.png)

我们需要根据其命名规则，指定相应的编译器，设置相应的`host`，`target`

关于`build`、`host`和`target`的含义可以参阅 [Cross-Compilation](https://www.gnu.org/software/automake/manual/html_node/Cross_002dCompilation.html) 这篇文档

- `build`: 编译该库所使用的平台，不设置的话，编译器会自动推测所在平台

- `host`: 编译出的库要运行在哪个平台上，不设置的话，默认为`build`值，但这样也就不再是交叉编译了

- `target`: 该库所处理的目标平台，不设置的话，默认为`host`值

多数`UNIX`平台会通过`CC`调用C语言编译器，而`CFLAGS`则是C语言编译器的编译选项，根据我们上文所说的命名规则可以发现，工具链中C语言编译器的命名规则为`${TRIPLE}${API}-clang`，假设我们要编译`arm64-v8a ABI`，`API 21`的库，则需要指定`CC`为`aarch64-linux-android21-clang`

至于`CFLAGS`这里就不多说了，可以自行查阅 [Clang编译器参数手册](https://clang.llvm.org/docs/ClangCommandLineReference.html) ，这里需要注意的是，必须要指定`--gcc-toolchain`和`--target`，否则编译会报错

然后就是`configure`的选项了，这里必须指定`--host`和`--sysroot`，`sysroot`表示使用这个值作为编译的头文件和库文件的查找目录，该目录结构如下

```
sysroot
└── usr
    ├── include
    └── lib
        ├── aarch64-linux-android
        ├── arm-linux-androideabi
        ├── i686-linux-android
        └── x86_64-linux-android
```

`--prefix`为编译后的安装路径，也就是编译产物的输出路径

`--enable-static`和`--enable-shared`选项表示生成静态库和动态库，大家可以根据情况自行选择

`nproc`是`Linux`下的一个命令，表示当前进程可用的`CPU`核数，一般`make`使用线程数为`CPU`核数就可以了，如果编译产生问题，可以尝试调小这个值

到这里基本上整个构建脚本就分析完了，大家调整完编译选项后保存，就可以执行命令`./build.sh`开始编译了

# FFmpeg

然后我们开始编译FFmpeg

```shell
git clone -b v5.0_compilable https://github.com/dreamgyf/FFmpeg.git
```

这个版本是从原`FFmpeg`镜像仓库的`n5.0`分支切出的，版本为`5.0`。其实我一开始用的是`5.1`版本，但当我解决了各种问题编译`OpenCV`到一半时，提示我`FFmpeg`的一些符号找不到，然后我去查了一下`OpenCV`的 Change Log ，发现它的最新版本`4.6.0`刚刚支持`FFmpeg 5.0`版本，无奈切到`5.0`重新编译

还是一样，先看编译脚本

```shell
# Linux 交叉编译 Android 库脚本
if [[ -z $ANDROID_NDK ]]; then
    echo 'Error: Can not find ANDROID_NDK path.'
    exit 1
fi

echo "ANDROID_NDK path: ${ANDROID_NDK}"

ROOT_PATH=`pwd`

OUTPUT_DIR="_output_"

mkdir ${OUTPUT_DIR}
cd ${OUTPUT_DIR}

OUTPUT_PATH=`pwd`

API=21
TOOLCHAIN=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64
# 编译出的x264库地址
X264_ANDROID_DIR=/home/dreamgyf/compile/x264/_output_/product

EXTRA_CONFIGURATIONS="--disable-stripping \
    --disable-ffmpeg \
    --disable-doc \
    --disable-appkit \
    --disable-avfoundation \
    --disable-coreimage \
    --disable-amf \
    --disable-audiotoolbox \
    --disable-cuda-llvm \
    --disable-cuvid \
    --disable-d3d11va \
    --disable-dxva2 \
    --disable-ffnvcodec \
    --disable-nvdec \
    --disable-nvenc \
    --disable-vdpau \
    --disable-videotoolbox"

function build {
    ABI=$1

    if [[ $ABI == "armeabi-v7a" ]]; then
        ARCH="arm"
        TRIPLE="armv7a-linux-androideabi"
    elif [[ $ABI == "arm64-v8a" ]]; then
        ARCH="arm64"
        TRIPLE="aarch64-linux-android"
    elif [[ $ABI == "x86" ]]; then
        ARCH="x86"
        TRIPLE="i686-linux-android"
    elif [[ $ABI == "x86-64" ]]; then
        ARCH="x86_64"
        TRIPLE="x86_64-linux-android"
    else
        echo "Unsupported ABI ${ABI}!"
        exit 1
    fi

    echo "Build ABI ${ABI}..."

    rm -rf ${ABI}
    mkdir ${ABI} && cd ${ABI}

    PREFIX=${OUTPUT_PATH}/product/$ABI

    export CC=$TOOLCHAIN/bin/${TRIPLE}${API}-clang
    export CFLAGS="-g -DANDROID -fdata-sections -ffunction-sections -funwind-tables -fstack-protector-strong -no-canonical-prefixes -D_FORTIFY_SOURCE=2 -Wformat -Werror=format-security  -O0 -DNDEBUG  -fPIC --gcc-toolchain=$TOOLCHAIN --target=${TRIPLE}${API}"

    ../../configure \
        --prefix=$PREFIX \
        --enable-cross-compile \
        --sysroot=$TOOLCHAIN/sysroot \
        --cc=$CC \
        --enable-static \
        --enable-shared \
        --disable-asm \
        --enable-gpl \
        --enable-libx264 \
        --extra-cflags="-I${X264_ANDROID_DIR}/${ABI}/include" \
        --extra-ldflags="-L${X264_ANDROID_DIR}/${ABI}/lib" \
        $EXTRA_CONFIGURATIONS

    make clean && make -j`nproc` && make install

    cd $PREFIX
    `$ROOT_PATH/ffmpeg-config-gen.sh ${X264_ANDROID_DIR}/${ABI}/lib/libx264.a`
    cd $OUTPUT_PATH
}

echo "Select arch:"
select arch in "armeabi-v7a" "arm64-v8a" "x86" "x86-64"
do
    build $arch
    break
done
```

这个脚本和`x264`的编译脚本基本相同，由于我们需要依赖`x264`库，所以我们要使刚刚编译出来的`x264`产物参与`FFmpeg`的编译，为此，需要将`X264_ANDROID_DIR`改成自己编译出来的`x264`产物路径

在`configure`选项中，我们需要`--enable-cross-compile`选项表示开启交叉编译，我们这里需要设置`--cc`选择C语言编译器，否则编译时会使用系统默认的编译器，`--disable-asm`选项我测试是必须要带上的，否则编译会报错，然后就是`--enable-libx264`开启`x264`依赖了，根据我在起因中说到的开源协议问题，所以`--enable-gpl`选项也要开启，最后需要指定`x264`的头文件和库文件目录，分别使用`--extra-cflags`和`--extra-ldflags`加上对应的参数

这里提一下，编译器会优先从`-I -L`两个参数指定的目录中去查找头文件和库文件，如果没找到的话再会从`sysroot`目录中查找

最后，我还写了一个`ffmpeg-config-gen.sh`脚本，它的作用是生成`ffmpeg-config.cmake`文件，用来给`OpenCV`编译提供`FFmpeg`依赖查找，这个等我们后面讲到`OpenCV`依赖`FFmpeg`的处理时再说

和`x264`一样，大家调整完编译选项后保存，就可以执行命令`./build.sh`开始编译了

# OpenCV

最后，我们开始编译`OpenCV`

```shell
git clone -b v4.6.0_compilable https://github.com/dreamgyf/opencv.git
```

这个版本是从原`OpenCV`仓库的`4.6.0`分支切出的，版本为`4.6.0`，是目前的最新版本。其实前面两个库的编译都挺顺利的，最麻烦的问题都出在`OpenCV`这里

我们还是先看编译脚本

```shell
# Linux 交叉编译 Android 库脚本
if [[ -z $ANDROID_NDK ]]; then
    echo 'Error: Can not find ANDROID_NDK path.'
    exit 1
fi

echo "ANDROID_NDK path: ${ANDROID_NDK}"

OUTPUT_DIR="_output_"

mkdir ${OUTPUT_DIR}
cd ${OUTPUT_DIR}

OUTPUT_PATH=`pwd`

API=21
TOOLCHAIN=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64
# 编译出的ffmpeg库地址
FFMPEG_ANDROID_DIR=/home/dreamgyf/compile/FFmpeg/_output_/product

EXTRA_ATTRS="-DWITH_CUDA=OFF \
    -DWITH_GTK=OFF \
    -DWITH_1394=OFF \
    -DWITH_GSTREAMER=OFF \
    -DWITH_LIBV4L=OFF \
    -DWITH_TIFF=OFF \
    -DBUILD_OPENEXR=OFF \
    -DWITH_OPENEXR=OFF \
    -DBUILD_opencv_ocl=OFF \
    -DWITH_OPENCL=OFF"

function build {
    ABI=$1

    if [[ $ABI == "armeabi-v7a" ]]; then
        ARCH="arm"
        TRIPLE="armv7a-linux-androideabi"
        TOOLCHAIN_NAME="arm-linux-androideabi"
    elif [[ $ABI == "arm64-v8a" ]]; then
        ARCH="arm64"
        TRIPLE="aarch64-linux-android"
        TOOLCHAIN_NAME="aarch64-linux-android"
    elif [[ $ABI == "x86" ]]; then
        ARCH="x86"
        TRIPLE="i686-linux-android"
        TOOLCHAIN_NAME="i686-linux-android"
    elif [[ $ABI == "x86-64" ]]; then
        ARCH="x86_64"
        TRIPLE="x86_64-linux-android"
        TOOLCHAIN_NAME="x86_64-linux-android"
    else
        echo "Unsupported ABI ${ABI}!"
        exit 1
    fi

    echo "Build ABI ${ABI}..."

    rm -rf ${ABI}
    mkdir ${ABI} && cd ${ABI}

    PREFIX=${OUTPUT_PATH}/product/$ABI

    cmake ../.. \
        -DCMAKE_INSTALL_PREFIX=$PREFIX \
        -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
        -DANDROID_ABI=$ABI \
        -DANDROID_NDK=$ANDROID_NDK \
        -DANDROID_PLATFORM="android-${API}" \
        -DANDROID_LINKER_FLAGS="-Wl,-rpath-link=$TOOLCHAIN/sysroot/usr/lib/$TOOLCHAIN_NAME/$API" \
        -DBUILD_ANDROID_PROJECTS=OFF \
        -DBUILD_ANDROID_EXAMPLES=OFF \
        -DBUILD_SHARED_LIBS=$BUILD_SHARED_LIBS \
        -DWITH_FFMPEG=ON \
        -DOPENCV_GENERATE_PKGCONFIG=ON \
        -DOPENCV_FFMPEG_USE_FIND_PACKAGE=ON \
        -DFFMPEG_DIR=${FFMPEG_ANDROID_DIR}/${ABI} \
        $EXTRA_ATTRS

    make clean && make -j`nproc` && make install

    cd ..
}

echo "Select arch:"
select arch in "armeabi-v7a" "arm64-v8a" "x86" "x86-64"
do
    echo "Select build static or shared libs:"
    select type in "static" "shared"
    do
        if [[ $type == "static" ]]; then
            BUILD_SHARED_LIBS=OFF
        elif [[ $type == "shared" ]]; then
            BUILD_SHARED_LIBS=ON
        else
            BUILD_SHARED_LIBS=OFF
        fi
        break
    done
    build $arch
    break
done
```

上面的准备工作和前面的几个脚本一样，不同的是，`OpenCV`并没有为我们准备`configure`脚本，所以这次我们使用`cmake`生成`Makefile`，再进行编译

既然使用`cmake`了，我们就可以不再像之前一样，自己指定编译器等工具链了，`NDK`为我们提供了交叉编译工具链`cmake`脚本`$ANDROID_NDK/build/cmake/android.toolchain.cmake`，我们只需要指定其为`CMAKE_TOOLCHAIN_FILE`，然后为其提供相关参数即可，具体的使用方式可以参考 [CMake](https://developer.android.com/ndk/guides/cmake?hl=zh-cn) 这篇文档。我们这里只需要提供最低限度的几个参数`ANDROID_ABI`、`ANDROID_NDK`、`ANDROID_PLATFORM`即可

如果需要编译`Android`示例工程的话，还需要在环境变量中设置`ANDROID_HOME`和`ANDROID_SDK`，我这里就直接使用`-DBUILD_ANDROID_PROJECTS=OFF`和`-DBUILD_ANDROID_EXAMPLES=OFF`将其关闭了

然后就是如何让`OpenCV`依赖我们编译的`FFmpeg`的问题了，到这一步我们就需要去它的`CMakeLists.txt`中看看它是怎样声明`FFmpeg`的了

打开`CMakeLists.txt`文件，搜索`FFMPEG`关键字，我们可以找到这一段代码

```cmake
if(WITH_FFMPEG OR HAVE_FFMPEG)
  if(OPENCV_FFMPEG_USE_FIND_PACKAGE)
    status("    FFMPEG:"       HAVE_FFMPEG         THEN "YES (find_package)"                       ELSE "NO (find_package)")
  elseif(WIN32)
    status("    FFMPEG:"       HAVE_FFMPEG         THEN "YES (prebuilt binaries)"                  ELSE NO)
  else()
    status("    FFMPEG:"       HAVE_FFMPEG         THEN YES ELSE NO)
  endif()
  status("      avcodec:"      FFMPEG_libavcodec_VERSION    THEN "YES (${FFMPEG_libavcodec_VERSION})"    ELSE NO)
  status("      avformat:"     FFMPEG_libavformat_VERSION   THEN "YES (${FFMPEG_libavformat_VERSION})"   ELSE NO)
  status("      avutil:"       FFMPEG_libavutil_VERSION     THEN "YES (${FFMPEG_libavutil_VERSION})"     ELSE NO)
  status("      swscale:"      FFMPEG_libswscale_VERSION    THEN "YES (${FFMPEG_libswscale_VERSION})"    ELSE NO)
  status("      avresample:"   FFMPEG_libavresample_VERSION THEN "YES (${FFMPEG_libavresample_VERSION})" ELSE NO)
endif()
```

我们可以发现，要想依赖`FFmpeg`，我们需要将`HAVE_FFMPEG`的值设为`true`，并且要指定`FFmpeg libs`的版本

我们再看到`OPENCV_FFMPEG_USE_FIND_PACKAGE`这个参数，表示通过`find_package`的方式寻找`FFmpeg`库

这里，我们其实有两种办法依赖`FFmpeg`库，一是通过`find_package`，二是通过`pkg-config`，我两种方式都尝试了后，觉得还是使用`find_package`这种方式比较容易，侵入性较小，使用`pkg-config`需要手动修改`OpenCV`检测`FFmpeg`的`cmake`文件源码，不优雅

接着我们去看`OpenCV`是如何检测`FFmpeg`是否存在的，这里我们需要找到`$OPENCV/modules/videoio/cmake/detect_ffmpeg.cmake`这个文件，在开头第一段代码中，我们就可以发现

```cmake
if(NOT HAVE_FFMPEG AND OPENCV_FFMPEG_USE_FIND_PACKAGE)
  if(OPENCV_FFMPEG_USE_FIND_PACKAGE STREQUAL "1" OR OPENCV_FFMPEG_USE_FIND_PACKAGE STREQUAL "ON")
    set(OPENCV_FFMPEG_USE_FIND_PACKAGE "FFMPEG")
  endif()
  find_package(${OPENCV_FFMPEG_USE_FIND_PACKAGE}) # Required components: AVCODEC AVFORMAT AVUTIL SWSCALE
  if(FFMPEG_FOUND OR FFmpeg_FOUND)
    set(HAVE_FFMPEG TRUE)
  endif()
endif()
```

如果`OPENCV_FFMPEG_USE_FIND_PACKAGE`选项被打开，则会使用`find_package(FFMPEG)`去查找这个库

`find_package(<PackageName>)`有两种模式，一种是`Module`模式，一种是`Config`模式

在`Module`模式中，`cmake`需要找到一个名为`Find<PackageName>.cmake`的文件，这个文件负责找到库所在路径，引入头文件和库文件。`cmake`会在两个地方查找这个文件，先是我们手动指定的`CMAKE_MODULE_PATH`目录，搜索不到再搜索`$CMAKE/share/cmake-<version>/Modules`目录

如果`Module`模式没找到相应文件，则会转为`Config`模式，在这个模式下，`cmake`需要找到`<lowercasePackageName>-config.cmake`或`<PackageName>Config.cmake`文件，通过这个文件找到库所在路径，引入头文件和库文件。`cmake`会优先在`<PackageName>_DIR`目录下搜索相应文件

关于`find_package`更详细的解释，可以去查看 [官方文档](https://cmake.org/cmake/help/latest/command/find_package.html)

我这里选用了`Config`模式，再结合之前在`CMakeLists.txt`和`detect_ffmpeg.cmake`中的内容，我们可以得出结论：

我们需要在构建脚本中设置`WITH_FFMPEG=ON`，`OPENCV_FFMPEG_USE_FIND_PACKAGE=ON`，`FFMPEG_DIR`并且`FFMPEG_DIR`目录下需要有`ffmpeg-config.cmake`或`FFMPEGConfig.cmake`文件

这里就衔接了上文，我为什么要写一个`ffmpeg-config-gen.sh`脚本，脚本的内容很简单，我们直接看它生成出来的`ffmpeg-config.cmake`文件

```cmake
set(FFMPEG_PATH "${CMAKE_CURRENT_LIST_DIR}")

set(FFMPEG_EXEC_DIR "${FFMPEG_PATH}/bin")
set(FFMPEG_LIBDIR "${FFMPEG_PATH}/lib")
set(FFMPEG_INCLUDE_DIRS "${FFMPEG_PATH}/include")

set(FFMPEG_LIBRARIES
    ${FFMPEG_LIBDIR}/libavformat.a
    ${FFMPEG_LIBDIR}/libavdevice.a
    ${FFMPEG_LIBDIR}/libavcodec.a
    ${FFMPEG_LIBDIR}/libavutil.a
    ${FFMPEG_LIBDIR}/libswscale.a
    ${FFMPEG_LIBDIR}/libswresample.a
    ${FFMPEG_LIBDIR}/libavfilter.a
    ${FFMPEG_LIBDIR}/libpostproc.a
    /home/dreamgyf/compile/x264/_output_/product/arm64-v8a/lib/libx264.a
    z
)

set(FFMPEG_libavformat_FOUND TRUE)
set(FFMPEG_libavdevice_FOUND TRUE)
set(FFMPEG_libavcodec_FOUND TRUE)
set(FFMPEG_libavutil_FOUND TRUE)
set(FFMPEG_libswscale_FOUND TRUE)
set(FFMPEG_libswresample_FOUND TRUE)
set(FFMPEG_libavfilter_FOUND TRUE)
set(FFMPEG_libpostproc_FOUND TRUE)

set(FFMPEG_libavcodec_VERSION 59.18.100)
set(FFMPEG_libavdevice_VERSION 59.4.100)
set(FFMPEG_libavfilter_VERSION 8.24.100)
set(FFMPEG_libavformat_VERSION 59.16.100)
set(FFMPEG_libavutil_VERSION 57.17.100)
set(FFMPEG_libpostproc_VERSION 56.3.100)
set(FFMPEG_libswresample_VERSION 4.3.100)
set(FFMPEG_libswscale_VERSION 6.4.100)

set(FFMPEG_FOUND TRUE)
set(FFMPEG_LIBS ${FFMPEG_LIBRARIES})
```

这里主要为`cmake`提供了三个变量

- `FFMPEG_INCLUDE_DIRS`：提供头文件目录

- `FFMPEG_LIBRARIES`：提供库文件链接

- `FFMPEG_FOUND`：告诉`cmake`找到了`FFmpeg`库

这里还有几个点要说，首先，`cmake`中的库文件链接顺序符合`gcc`链接顺序规则，所以说库的书写顺序也是有严格要求的，被依赖的库要放在依赖它的库的后面，正如这个文件，`FFmpeg`需要依赖`x264`，所以我需要将`x264`放在所有`FFmpeg`库的最后面

`FFmpeg`需要依赖`zlib`库，所以我在后面增加了一个`z`表示依赖`zlib`库

`FFmpeg`这些库的版本定义是从`$FFMPEG_PRODUCT/$ABI/lib/pkgconfig`目录下各个文件读出来的

![pkgconfig](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91OpenCV%2BFFmpeg%2Bx264%E7%9A%84%E8%89%B0%E9%9A%BE%E5%8E%86%E7%A8%8B_pkgconfig.png)

![version](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91OpenCV%2BFFmpeg%2Bx264%E7%9A%84%E8%89%B0%E9%9A%BE%E5%8E%86%E7%A8%8B_version.png)

`ffmpeg-config.cmake`文件写完，我们再回过头来看一下`detect_ffmpeg.cmake`

```cmake
if(NOT HAVE_FFMPEG AND OPENCV_FFMPEG_USE_FIND_PACKAGE)
  if(OPENCV_FFMPEG_USE_FIND_PACKAGE STREQUAL "1" OR OPENCV_FFMPEG_USE_FIND_PACKAGE STREQUAL "ON")
    set(OPENCV_FFMPEG_USE_FIND_PACKAGE "FFMPEG")
  endif()
  find_package(${OPENCV_FFMPEG_USE_FIND_PACKAGE}) # Required components: AVCODEC AVFORMAT AVUTIL SWSCALE
  if(FFMPEG_FOUND OR FFmpeg_FOUND)
    set(HAVE_FFMPEG TRUE)
  endif()
endif()
```

可以看到最后的 if 中，如果`FFMPEG_FOUND`为`true`，则设置`HAVE_FFMPEG`为`true`，正好对应了我们在`ffmpeg-config.cmake`中的行为，这下，`CMakeLists.txt`就可以找到我们的`FFmpeg`库了

这里还有一点，`detect_ffmpeg.cmake`中有一段用来测试的代码

```cmake
if(HAVE_FFMPEG AND NOT HAVE_FFMPEG_WRAPPER AND NOT OPENCV_FFMPEG_SKIP_BUILD_CHECK)
  try_compile(__VALID_FFMPEG
      "${OpenCV_BINARY_DIR}"
      "${OpenCV_SOURCE_DIR}/cmake/checks/ffmpeg_test.cpp"
      CMAKE_FLAGS "-DINCLUDE_DIRECTORIES:STRING=${FFMPEG_INCLUDE_DIRS}"
                  "-DLINK_LIBRARIES:STRING=${FFMPEG_LIBRARIES}"
      OUTPUT_VARIABLE TRY_OUT
  )
  if(NOT __VALID_FFMPEG)
    message(FATAL_ERROR "FFMPEG: test check build log:\n${TRY_OUT}")
    message(STATUS "WARNING: Can't build ffmpeg test code")
    set(HAVE_FFMPEG FALSE)
  endif()
endif()
```

其中的`message(FATAL_ERROR "FFMPEG: test check build log:\n${TRY_OUT}")`原本是被注释了的，我强烈建议各位将其打开，这样如果哪里有误，一开始就可以报错并附带详细信息，免得到时候编到一半才报错，浪费时间

到这里，我本以为万事大吉了，于是开始编译，这里我使用了`BUILD_SHARED_LIBS=ON`选项编译动态库，`armeabi-v7a`顺利编译通过，但当`arm64-v8a`编译到一半时突然报错，提示`libz.so, needed by ../../lib/arm64-v8a/libopencv_core.so, not found (try using -rpath or -rpath-link)`

![rpath_error](https://raw.githubusercontent.com/dreamgyf/ImageStorage/master/Android%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91OpenCV%2BFFmpeg%2Bx264%E7%9A%84%E8%89%B0%E9%9A%BE%E5%8E%86%E7%A8%8B_rpath_error.png)

我观察了一下`NDK`目录结构，发现`libz.so`动态库文件可以在`$TOOLCHAIN/sysroot/usr/lib/$TOOLCHAIN_NAME/$API`下找到，需要注意的是，这里的`TOOLCHAIN_NAME`和`TRIPLE`很相似，但在`armeabi-v7a`情况下又有些细微的不同，所以我又新定义了这个变量

然后我开始尝试加入`-rpath-link`选项，首先，我尝试添加了一项`cmake`选项`CMAKE_SHARED_LINKER_FLAGS="-Wl,-rpath-link=$TOOLCHAIN/sysroot/usr/lib/$TOOLCHAIN_NAME/$API"`，发现，虽然在编译开头的输出中可以看出，这个参数确实被加上生效了，但在编译到同样的地方时，仍然会报相同的错误，这里我不太清楚，难道参数的顺序也会对编译造成影响吗？

于是我去查看了`android.toolchain.cmake`文件，看他是怎么添加这些选项的，发现了这么一行

```cmake
set(CMAKE_SHARED_LINKER_FLAGS "${ANDROID_LINKER_FLAGS} ${CMAKE_SHARED_LINKER_FLAGS}")
```

于是我在这行代码前加了这么一行

```cmake
list(APPEND ANDROID_LINKER_FLAGS -Wl,-rpath-link=${ANDROID_TOOLCHAIN_ROOT}/sysroot/usr/lib/${ANDROID_TOOLCHAIN_NAME}/${ANDROID_PLATFORM_LEVEL})
```

让`-rpath-link`这个选项提前一点，果不其然，编译顺利通过了，但这样做有点麻烦，还得改`NDK`里的配置，于是我在构建脚本里加了一个参数`ANDROID_LINKER_FLAGS="-Wl,-rpath-link=$TOOLCHAIN/sysroot/usr/lib/$TOOLCHAIN_NAME/$API"`，这样的话，`-rpath-link`选项会被提到`Linker flags`的最前面，经过测试，这样也可以编译通过，于是`OpenCV`的编译脚本也就这么完成了

当然这里还剩一个疑点，为什么不加`-rpath-link`的时候，`arm64-v8a`编译报错但`armeabi-v7a`却编译通过，希望有大佬可以指点一下

# FreeType

我的App中还用到了`FreeType`库渲染字体，在这里顺便也把它的编译方式放出来吧

直接去 [FreeType](https://github.com/dreamgyf/freetype/releases/tag/v2.12.1_compilable) 这里下载我编译好的版本或者源码，根据我写的步骤进行编译就可以了

# 在Android中使用

在Android中使用时需要注意，如果你使用静态库的方式的话，需要将`OpenCV`编译出来的第三方库也加入到链接中，放在`OpenCV`的后面，另外`FFmpeg`还需要`mediandk`和`zlib`这两个依赖，具体可以参考下面的代码

```cmake
target_link_libraries(
        textvideo

        freetype

        # opencv
        opencv_videoio
        opencv_photo
        opencv_highgui
        opencv_imgproc
        opencv_imgcodecs
        opencv_dnn
        opencv_core

        # ffmpeg
        ffmpeg_avformat
        ffmpeg_avdevice
        ffmpeg_avcodec
        ffmpeg_avutil
        ffmpeg_swscale
        ffmpeg_swresample
        ffmpeg_avfilter

        # ffmpeg依赖
        mediandk
        z

        # x264
        x264

        # opencv第三方支持库
        ade
        cpufeatures
        ittnotify
        libjpeg-turbo
        libopenjp2
        libpng
        libprotobuf
        libwebp
        quirc
        tegra_hal

        # android jni库
        jnigraphics
        android
        log)
```

# 总结

虽然我这篇文章写的看起来编译的过程很简单，根本不像标题所说的那么艰难，但实际上我前前后后弄了大概有一个多星期才真正完整编出可用版本，前前后后编译失败了不说一百次也有几十次，对我这种不懂c语言编译的简直是折磨。因为我是在全部弄完后才开始写的文章，所以基本上坑都踩的差不多了，其中有些坑印象也没那么清楚了，我也没那么多精力再去复现出那些坑了，怎么说呢，能成功就万事大吉吧 😭