---
layout: post
title: "Android 平台用 gprof 给 Cocos2d-x 做性能分析"
date: 2015-04-17 11:27:52 +0800
comments: true
categories: [Cocos2d-x Android]
---
###gprof

在 iOS 平台下我们可以用 Xcode 自带的 Profile 工具来测试我们程序的性能，那在 Android 平台下面要怎么搞呢？答案就是`gprof`。什么是 gprof 呢？引用 [Wiki](http://en.wikipedia.org/wiki/Gprof) 的解释：

>Gprof is a performance analysis tool for Unix applications. It uses a hybrid of instrumentation and sampling[1] and was created as extended version of the older "prof" tool. Unlike prof, gprof is capable of limited call graph collecting and printing.

因为 Android 本来就是基于 Linux 的，所以这里用 gprof 来做性能测试是没什么问题的。不过需要注意的是，这里所说的性能测试是针对 NDK 编译的 C++ 代码的。就想 Cocos2d-x 这样的 C++ 实现的游戏引擎就可以通过 gprof 来分析。下面我们来说说搞法。

###环境

我是 Mac OS X 下，这里要做性能分析的 Cocos2d-x 项目是基于 Cocos2d-x 3.2 引擎，项目本身是基于 Lua 脚本编写的。其实这些都无关紧要，只不过是编译出的 so 文件有所不同罢了。只要是 NDK 的代码都可以用 gprof 来做性能分析的。

###android-ndk-profiler

要想生成 gprof 的性能分析报告，我们优先要把一个叫做 `android-ndk-profiler` 的模块集成到我们的项目中。android-ndk-profiler 模块的源代码在 [GitHub](https://github.com/richq/android-ndk-profiler) 上面，首先要把模块代码 clone 下来

```
git clone git@github.com:richq/android-ndk-profiler.git
```

android-ndk-profiler 的项目自带了一篇[文档说明](https://github.com/richq/android-ndk-profiler/blob/master/docs/Usage.md)教授如何集成和使用，但是写的比较简单，我来详细的说一下。android-ndk-profiler 项目 clone 下来后进到项目目录可以看到如下结构

```
 |-docs
 |-example
 |-jni
 |-test
``` 
 
 而我们需要的就是`jni`这个目录下面的文件。
 
 
###集成 android-ndk-profiler 到 Cocos2d-x 项目

######拷贝文件

来到我们自己的 Cocos2d-x 项目目录中，新建一个叫做 `android-ndk-profiler` 的文件夹，将刚刚克隆的 android-ndk-profile 模块的 jni 目录中的所有文件拷贝到我们刚刚建立的文件夹中。

######编辑 Android.mk 文件

打开 `proj.android/jin/Android.mk` 文件，加入如下代码：

```
#*注意* YOUR_ANDROID_NDK_PROFILER_PATH 是你 cocos2d-x 项目中 android-ndk-profiler 目录的位置

$(call import-add-path,$(YOUR_ANDROID_NDK_PROFILER_PATH))

# 加入头文件
LOCAL_C_INCLUDES += $(YOUR_ANDROID_NDK_PROFILER_PATH)

APP_DEBUG := $(strip $(NDK_DEBUG))

# 如果是 Debug 模式，则引入 android-ndk-profiler
ifeq ($(APP_DEBUG),1)
	LOCAL_CFLAGS := -pg
	LOCAL_STATIC_LIBRARIES += android-ndk-profiler
endif

ifeq ($(APP_DEBUG),1)
	$(call import-module,YOUR_ANDROID_NDK_PROFILER_PATH)
endif
```

这里只解释两点。

`LOCAL_CFLAGS := -pg` 通过在编译使用 -pg 编译和链接选项，gcc 在你应用程序的每个函数中都加入了一个名为mcount ( or  “_mcount”  , or  “__mcount” , 依赖于编译器或操作系统) 的函数，也就是说你的应用程序里的每一个函数都会调用 mcount, 而 mcount 会在内存中保存一张函数调用图，并通过函数调用堆栈的形式查找子函数和父函数的地址。这张调用图也保存了所有与函数相关的调用时间，调用次数等等的所有信息。

`APP_DEBUG` 这里增加了一个判断，只有当以 `NDK_DEBUG=1` 的 Debug 模式编译 NDK 代码的时候才开启 android-ndk-profiler 分析功能，这样保证我们出 Release 版本的时候不引入性能分析。

###使用 android-ndk-profiler

以 Debug 模式重新编译一下项目代码：

```
cd proj.android
ndk-build NDK_DEBUG=1
```

如果编译成功那么说明 android-ndk-profiler 已经成功集成到我们的 Cocos2d-x 项目中了，集成的过程非常简单，同样，android-ndk-profiler 的使用也非常的方便。

######编辑 AppDelegate.cpp 文件

只需要引入一个头文件，添加两个函数调用即可

```
// 引入头文件
#if (COCOS2D_DEBUG>0 && CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
#include "prof.h"
#endif

bool AppDelegate::applicationDidFinishLaunching()
{
#if (COCOS2D_DEBUG>0 && CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
	monstartup("libcocos2dlua.so");
#endif
	// 其他已有逻辑代码......
}

void AppDelegate::applicationDidEnterBackground()
{
	// 其他已有逻辑代码......
#if (COCOS2D_DEBUG>0 && CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
    moncleanup();
#endif
}

```

这里只需要注意两点。

`AndroidManifest.xml` 因为要生成性能分析报告，所以要赋予你的 Android 程序 WRITE_EXTERNAL_STORAGE 权限，即

```
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

`libcocos2dlua.so` 这个 so 文件会根据你的 Cocos2d-x 项目的类型不同名字上会有所不同，比如我们是 Lua 项目，所以 NDK 编译生成的 so 文件就叫 libcocos2dlua.so ，具体的文件名请自行到 proj.android/libs/armeabi 目录下查看。

再次以 Debug 模式重新编译一下项目代码，如果没有错误，那么大功就告成了。

######生成 gmon.out 性能分析报告

项目编译完成后生成 apk 文件，将 apk 文件安装到 Android 设备上。通过上一小节我们对 AppDelegate.cpp 文件的修改不难看出，当程序在 Android 设备上运行的时候，调用了 `monstartup` 函数开始性能分析，当程序退到后台时调用了 `moncleanup` 函数生成性能分析报告。性能分析报告文件默认存储到 Android 设备的 /sdcard/gmon.out 位置，我们用 adb 工具可以把文件拉到电脑上面。

```
adb pull /sdcard/gmon.out .
```

当然官方文档里面也提了，如果想要自定义性能分析报告存放的位置，可以在调用 `moncleanup` 函数前指定要保存的位置。

```
setenv("CPUPROFILE", "/data/data/com.example.application/files/gmon.out", 1);
moncleanup();
```

###解读性能分析报告 gmon.out

生成的性能分析告报 gmon.out 是不能直接通过文本编辑器打开来读的，它是个二进制文件，需要专门的工具来生成可读的文本文件。这个工具在 NDK 中已经提供了，以我使用的 android-ndk-r10d 为例：

```
cd android-ndk-r10d/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/

./arm-linux-androideabi-gprof \
	你的项目路径/proj.android/obj/local/armeabi/libcocos2dlua.so\
	你的gmon.out存放路径/gmon.out > gmon.txt
```

这里只解释一点。

`libcocos2dlua.so` 细心的读者发现这里使用的 so 文件并不是之前的那个放在 proj.android/libs/armeabi/libcocos2dlua.so 下面的那个 so 文件。这是因为最终随 apk 一起打包的那个 libcocos2dlua.so 文件（也就是 proj.android/libs/armeabi 目录下的）是不包含**符号表**的，而存放在 proj.android/obj/local/armeabi 目录下的是带**符号表**的版本。而什么是**符号表**，这是一个编译链接中的概念，请自行 Google 一下，或者读一读[《程序员的自我修养》这本书](http://book.douban.com/subject/3652388/)，再次强烈推荐这本书。

###gmon.txt 解读

我节选一下生成的 gmon.txt 的两处比较重要的部分来看

```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 19.14      1.16     1.16                             png_read_filter_row_paeth_multibyte_pixel
 15.35      2.09     0.93                             cocos2d::Image::premultipliedAlpha()
 14.36      2.96     0.87                             cocos2d::Texture2D::convertRGBA8888ToRGBA4444(unsigned char const*, int, unsigned char*)
  7.10      3.39     0.43                             profCount
  3.96      3.63     0.24                             png_read_filter_row_up
  3.30      3.83     0.20                             llex
  3.14      4.02     0.19                             png_read_filter_row_sub
  2.81      4.19     0.17                             __gnu_mcount_nc
  
  ......
  
  
		     Call graph (explanation follows)


granularity: each sample hit covers 2 byte(s) for 0.17% of 5.99 seconds

index % time    self  children    called     name
                                                 <spontaneous>
[1]     19.4    1.16    0.00                 png_read_filter_row_paeth_multibyte_pixel [1]
-----------------------------------------------
                                                 <spontaneous>
[2]     15.5    0.93    0.00                 cocos2d::Image::premultipliedAlpha() [2]
-----------------------------------------------
                                                 <spontaneous>
[3]     14.5    0.87    0.00                 cocos2d::Texture2D::convertRGBA8888ToRGBA4444(unsigned char const*, int, unsigned char*) [3]
-----------------------------------------------
                                                 <spontaneous>
[4]      7.2    0.43    0.00                 profCount [4]
-----------------------------------------------
                                                 <spontaneous>
[5]      4.0    0.24    0.00                 png_read_filter_row_up [5]
-----------------------------------------------

```

简单解释一下含义

```
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name
 函数     程序       函数       函数    函数     函数      函数名
 消耗     累计       本身       调用    平均     平均
 时间     执行       执行       次数    执行     执行
 占程     时间       时间              时间     时间
 序运                                 (不      (包
 行时                                  包       括
 间的                                  括       被
 百分                                  被       调
 比                                    调       用
                                      用       时
                                      时       间)
                                      间)


		     Call graph (explanation follows)


granularity: each sample hit covers 2 byte(s) for 0.17% of 5.99 seconds

index % time    self  children    called     name
 索引    函数     函数   函数的       被调用      函数名
 值      执行   	 本身   子函数        次数
        时间     执行    执行
        占程     时间    时间
        序运
        行时
        间百
        分比
        

```