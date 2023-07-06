---
layout: post
tags: FFmpeg ndk
date: 2020-10-13
thumbnail: assets/img/e003.webp
title: 编译最新版FFmpeg for Android NDK(r21)
published: true
---

这里介绍在Mac下使用最新版NDK(r21)编译最新版的FFmpeg(4.3.1)，在NDK r17之后弃用了gcc，改用clang进行编译，因此最新版本NDK主要解决用clang配置编译ffmpeg。

<!--more-->

### 准备
- NDK
下载NDK(r21)
解压NDK
（这一步网上资料较多不再赘述）

- FFmpeg：
下载官网最新版本的FFmpeg(4.3.1)：[https://www.ffmpeg.org/download.html#releases](https://www.ffmpeg.org/download.html#releases)

![FFmpeg文件目录](https://raw.githubusercontent.com/ruilin/blog/master/assets/img/e001.webp)

### 修改编译配置
> 默认配置是编译生成Linux环境下的库，需要修改为生成Android环境下的so

用文本编辑器（比如Sublime text）打开ffmpeg文件夹下的configure文件，找到以下配置进行替换：
```
# 注释掉默认configure 文件中的配置：
# SLIBNAME_WITH_MAJOR='$(SLIBNAME).$(LIBMAJOR)'
# LIB_INSTALL_EXTRA_CMD='$$(RANLIB) "$(LIBDIR)/$(LIBNAME)"'
# SLIB_INSTALL_NAME='$(SLIBNAME_WITH_VERSION)'
# SLIB_INSTALL_LINKS='$(SLIBNAME_WITH_MAJOR) $(SLIBNAME)'

# 替换为：
SLIBNAME_WITH_MAJOR='$(SLIBPREF)$(FULLNAME)-$(LIBMAJOR)$(SLIBSUF)'
LIB_INSTALL_EXTRA_CMD='?(RANLIB) "$(LIBDIR)/$(LIBNAME)"'
SLIB_INSTALL_NAME='$(SLIBNAME_WITH_MAJOR)'
SLIB_INSTALL_LINKS='$(SLIBNAME)'
```
### 创建编译脚本
- 打开终端，cd到ffmpeg目录下，执行以下命令创建脚本文件：
```
touch build_android.sh
```
- 用vim打开文件：
```
vim build_android.sh
```

- 复制以下shell脚本到 build_android.sh：
（注意：NDK需要修改成自己的路径）

```
#!/bin/bash
# 以下路径需要修改成自己的NDK目录
TOOLCHAIN=/Users/xxx/Library/Android/sdk/ndk/21.3.6528147/toolchains/llvm/prebuilt/darwin-x86_64/
# 最低支持的android sdk版本
API=29

function build_android
{
# 打印
echo "Compiling FFmpeg for $CPU"
# 调用同级目录下的configure文件
./configure \
# 指定输出目录
    --prefix=$PREFIX \
# 各种配置项，想详细了解的可以打开configure文件找到Help options:查看
    --disable-neon \
    --disable-hwaccels \
    --disable-gpl \
    --disable-postproc \
    --enable-shared \
    --enable-jni \
    --disable-mediacodec \
    --disable-decoder=h264_mediacodec \
    --disable-static \
    --disable-doc \
    --disable-ffmpeg \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-avdevice \
    --disable-doc \
    --disable-symver \
    --cross-prefix=$CROSS_PREFIX \
    --target-os=android \
    --arch=$ARCH \
    --cpu=$CPU \
    --cc=$CC
    --cxx=$CXX
    --enable-cross-compile \
    --sysroot=$SYSROOT \
    --extra-cflags="-Os -fpic $OPTIMIZE_CFLAGS" \
    --extra-ldflags="$ADDI_LDFLAGS" \
    $ADDITIONAL_CONFIGURE_FLAG
make clean
make
make install
echo "The Compilation of FFmpeg for $CPU is completed"
}

#armv8-a
ARCH=arm64
CPU=armv8-a
# r21版本的ndk中所有的编译器都在/ndk/21.3.6528147/toolchains/llvm/prebuilt/darwin-x86_64/目录下（clang）
CC=$TOOLCHAIN/bin/aarch64-linux-android$API-clang
CXX=$TOOLCHAIN/bin/aarch64-linux-android$API-clang++
# NDK头文件环境
SYSROOT=$TOOLCHAIN/sysroot
CROSS_PREFIX=$TOOLCHAIN/bin/aarch64-linux-android-
# so输出路径
PREFIX=$(pwd)/android/$CPU
OPTIMIZE_CFLAGS="-march=$CPU"
build_android

# 交叉编译工具目录,对应关系如下
# armv8a -> arm64 -> aarch64-linux-android-
# armv7a -> arm -> arm-linux-androideabi-
# x86 -> x86 -> i686-linux-android-
# x86_64 -> x86_64 -> x86_64-linux-android-

# CPU架构
#armv7-a
ARCH=arm
CPU=armv7-a
CC=$TOOLCHAIN/bin/armv7a-linux-androideabi$API-clang
CXX=$TOOLCHAIN/bin/armv7a-linux-androideabi$API-clang++
SYSROOT=$TOOLCHAIN/sysroot
CROSS_PREFIX=$TOOLCHAIN/bin/arm-linux-androideabi-
PREFIX=$(pwd)/android/$CPU
OPTIMIZE_CFLAGS="-mfloat-abi=softfp -mfpu=vfp -marm -march=$CPU "
build_android

#x86
ARCH=x86
CPU=x86
CC=$TOOLCHAIN/bin/i686-linux-android$API-clang
CXX=$TOOLCHAIN/bin/i686-linux-android$API-clang++
SYSROOT=$TOOLCHAIN/sysroot
CROSS_PREFIX=$TOOLCHAIN/bin/i686-linux-android-
PREFIX=$(pwd)/android/$CPU
OPTIMIZE_CFLAGS="-march=i686 -mtune=intel -mssse3 -mfpmath=sse -m32"
build_android

#x86_64
ARCH=x86_64
CPU=x86-64
CC=$TOOLCHAIN/bin/x86_64-linux-android$API-clang
CXX=$TOOLCHAIN/bin/x86_64-linux-android$API-clang++
SYSROOT=$TOOLCHAIN/sysroot
CROSS_PREFIX=$TOOLCHAIN/bin/x86_64-linux-android-
PREFIX=$(pwd)/android/$CPU
OPTIMIZE_CFLAGS="-march=$CPU -msse4.2 -mpopcnt -m64 -mtune=intel"
# 方法调用
build_android
```

- 输入`:wq`保存

### 开始编译
- 先给脚本添加权限：
```
chmod 777 ./build_android.sh
```
- 然后就可以运行脚本编译了：
```
./build_android.sh
```

### 编译完成
![编译完成](https://raw.githubusercontent.com/ruilin/blog/master/assets/img/e002.webp)

编译完成后将输出文件

![so输出](https://raw.githubusercontent.com/ruilin/blog/master/assets/img/e003.webp)


### 常见问题
- x86版本编译问题
如果编译完后发现没有输出x86 so，检查是否有输出以下错误信息：
```
nasm/yasm not found or too old. Use --disable-x86asm for a crippled build.
```
原因是编译x86依赖汇编器nasm或者yasm没有安装。
因此需要先安装好汇编器，这里我们选择其一进行安装
安装方法：`brew install yasm`（通过homebrew 进行安装）
安装成功再重新编译即可。

- Android 加载x86 so问题
在运行时加载so报错：
```
java.lang.UnsatisfiedLinkError: dlopen failed: /data/app/xxx.xxx.xxx-1/lib/x86/libavcodec.so: has text relocations
```
原因是android 6.0之后，系统做了限制。[Android 6.0 changes](https://developer.android.com/about/versions/marshmallow/android-6.0-changes.html)
> On previous versions of Android, if your app requested the system to load a shared library with text relocations, the system displayed a warning but still allowed the library to be loaded. Beginning in this release, the system rejects this library if your app's target SDK version is 23 or higher. To help you detect if a library failed to load, your app should log the dlopen(3) failure, and include the problem description text that the dlerror(3) call returns. To learn more about handling text relocations, see this guide.

解决方法：需要在编译选项中增加参数`--disable-asm`
![增加参数](https://raw.githubusercontent.com/ruilin/blog/master/assets/img/e004.webp)
如果出现以下错误
```
dlopen failed: cannot locate symbol "iconv_open"，”iconv_close“
```
只能降低API版本，修改build_android.sh：`API=26`




[参考]

[1] https://yesimroy.gitbooks.io/android-note/content/compile_ffmpeg_for_android.html

[2] https://stackoverflow.com/questions/57681336/build-ffmpeg-4-2-with-android-ndk-r20

[3] https://www.jianshu.com/p/feab970fd74c

[4] https://juejin.im/post/6844903945496690696

[5] https://blog.csdn.net/marco_0631/article/details/73292199

[6] https://www.jianshu.com/p/fd938a51b01f

[7] https://github.com/tianshaokai/ffmpegbuild