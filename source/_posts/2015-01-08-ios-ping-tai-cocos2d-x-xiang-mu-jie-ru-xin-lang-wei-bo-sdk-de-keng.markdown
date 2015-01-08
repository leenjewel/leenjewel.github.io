---
layout: post
title: "iOS 平台 Cocos2d-x 项目接入新浪微博 SDK 的坑"
date: 2015-01-08 14:48:45 +0800
comments: true
categories: [cocos2d-x]
---

最近在做一个 iOS 的 cocos2d-x 项目接入新浪微博 SDK 的时候被“坑”了，最后终于顺利的解决了。发现网上也有不少人遇到一样的问题，但是能找到的数量有限的解决办法写得都不详细，很难让人理解，我来深入的写一写。

###我的开发环境

- Mac OS X 10.10.1

- Xcode 6.1.1 (6A2008a)

- Cocos2d-x 3.2

- 新浪微博 SDK for iOS 2015 年 1 月 5 日从 [github](https://github.com/sinaweibosdk/weibo_ios_sdk) clone 的版本

###遇到的问题

根据新浪微博 SDK 附带的文档接入项目后，在模拟器运行项目，在调用注册方法时发生崩溃。注册方法代码：

```
[WeiboSDK registerApp: @"xxxxxxxx"];
```

崩溃信息打印如下：

```
[__NSDictionaryM weibosdk_WBSDKJSONString] : unrecognized selector sent to instance 0x170255780
```

###解决问题遇到的阻碍

新浪微博 SDK 附带的文档中有这么一个说明：

> 在工程中引入静态库之后,需要在编译时添加   –ObjC   编译选项,避免静态库中类 加载   不全造成程序崩溃。方法:程序   Target->Buid   Settings->Linking   下   Other   Linker  Flags   项添加-ObjC


在网上看到遇到同样崩溃错误的人有提到在编译时添加 `-all_load` 编译选项时也可以解决问题。方法也是在   Target->Buid   Settings->Linking   下   Other   Linker  Flags   项添加`-all_load`。

无独有偶，我在打开新浪微博 SDK 附带的 Demo 项目时发现这个项目的编译选项也是`-all_load`而不是它自己文档所提示的~~`-ObjC`~~。而且在同样的开发环境下，我的 cocos2d-x 项目会崩溃，但是新浪微博 SDK 附带的 Demo 可以正常工作，想必上述两个解决方案**应该是正解**

但是在给自己的 cocos2d-x 项目添加了编译选项后，再次编译运行就发生了错误，错误信息如下：

```
Undefined symbols for architecture i386:
  "_GCControllerDidConnectNotification", referenced from:
      -[GCControllerConnectionEventHandler observerConnection:disconnection:] in libcocos2dx iOS.a(CCController-iOS.o)
  "_GCControllerDidDisconnectNotification", referenced from:
      -[GCControllerConnectionEventHandler observerConnection:disconnection:] in libcocos2dx iOS.a(CCController-iOS.o)
  "_OBJC_CLASS_$_GCController", referenced from:
      objc-class-ref in libcocos2dx iOS.a(CCController-iOS.o)
     (maybe you meant: _OBJC_CLASS_$_GCControllerConnectionEventHandler)
```

无论是设置成~~`-ObjC`~~还是~~`-all_load`~~编译都会失败，都会报上述**找不到符号**的链接错误。

###正确的解决办法

这里先给出正确的解决办法再谈谈为什么要这么做。正确的做法还是设置 Other Linker Flags 这个编译选项，只不过即不用用~~`-ObjC`~~也不能用~~`-all_load`~~，而是要用**`-force_load path/to/your/libWeiboSDK.a`**，后面跟的是新浪微博 SDK 静态链接库的确切位置。

###这一切是为什么？

###从编译链接说起

这里不打算过多的介绍编译链接相关的只是，但是强烈推荐一本书[《程序员的自我修养》](http://book.douban.com/subject/3652388/)，光看正标题你可能会担心这是本没什么“正经”内容的书，至少我当初第一次看到这书名的时候就是这么认为的，但是我错了，这本书的副标题是**链接、装载与库**。相信我，看过这本书 N 遍之后你自会对程序从源代码编译链接到生成二进制程序的原理和过程有一个非常透彻的理解，并且更重要的是看过这本书 N 遍之后你会上升几个层次。

言归正传，一个工程的源代码最终变成二进制的可执行程序、动态链接库或静态链接库要经历这么几个过程：

```
源代码 ==[编译器]==》 汇编码 ==[汇编器]==》 对象文件 ==[链接器]==》 可执行程序、动态链接库或静态链接库
```

###再说说符号是什么？

通俗的讲，我们在源码中写的`全局变量名`、`函数名`或`类名`在生成的`*.o`对象文件中都叫做符号，存在一个叫做`符号表`的地方。

举个例子：我们在`a.c`文件中写了一个函数叫`foo()`，然后在`main.c`文件中调用了`foo()`函数，在将源码编译生成的对象文件中`a.o`对象文件中的符号表里保存着`foo()`函数符号，并通过该符号可以定位到`a.o`文件中关于`foo()`方法的具体实现代码。

链接器在链接生成最终的二进制程序的时候会发现`main.o`对象文件中引用了符号`foo()`，而`foo()`符号并没有在`main.o`文件中定义，所以不会存在与`main.o`对象文件的符号表中，于是链接器就开始检查其他对象文件，当检查到`a.o`文件中定义了符号`foo()`，于是就将`a.o`对象文件链接进来。这样就确保了在`main.c`中能够正常调用`a.c`中实现的`foo()`方法了。

###libWeiboSDK.a 静态链接库里有什么？

Unix 的静态链接库没什么神秘的，它就是个**压缩包**，和平时比较常见的 zip 或 rar 之类的压缩包一样，只不过人家是用一个叫 ar 的压缩工具压缩的而已。所以我们给它解压缩一下，看看它里面都有什么。既然是用 ar 压缩的，解压自然也要用 ar 这个工具。在命令行执行：

```
ar -x lieWeiboSDK.a
```

结果报错了：

```
ar: libWeiboSDK.a is a fat file (use libtool(1) or lipo(1) and ar(1) on it)
ar: libWeiboSDK.a: Inappropriate file type or format
```

这里先解释一下它为什么这么肥(fat)。在做 iOS 开发时我们都知道可以用模拟器和真机来测试我们的项目，但是这两个平台的架构是不一样的，模拟器是 i386 x86_64 架构的，而我们的设备是 armv7 arm64 架构的。当在制作静态链接库的时候也要针对不同的架构制作出针对真机和模拟器的两个静态链接库，而当我们想在自己的项目中使用静态链接库的时候，如果在模拟器上运行我们要用针对模拟器的静态库版本，用真机设备测试的时候还要切换到针对真机的静态链接库，这样一来非常的麻烦。

前面说过了静态链接库就是个**压缩包**，那么我们是否能将这两个静态链接库压缩成一个静态链接库这样就可以同时支持模拟器和真机设备两种架构了呢？答案是肯定的。比如我们手头有一个静态链接库的两个架构版本：`libXXX.i386_x86_64.a`和`libXXX.armv7_arm64.a`，那么我们可以通过如下命令来生成一个统一的静态链接库：

```
lipo -create libXXX.i386_x86_64.a libXXX.armv7_arm64.a -output libXXX.a
```

这样我们就得到了一个统一版本的静态库`libXXX.a`，它的好处是同时支持模拟器架构和真机设备架构，缺点是它的体积变大了，也就是说它很**肥(fat)**。

而`libWeiboSDK.a`就是这么一个合体后的静态库，我们照样可以通过命令来验证这一点：

```
lipo -info libWeiboSDK.a
```

这个命令会输出：

```
Architectures in the fat file: libWeiboSDK.a are: armv7 arm64 i386 x86_64
```

既然是个胖子，那我们就要先给它瘦身才能解压。我们随便从里面抽出一个架构的静态链接库来，瘦身命令是：

```
lipo -thin i386 libWeiboSDK.a -output libWeiboSDK.i386.a
```

这样我们就把针对 i386 平台的新浪微博 SDK 静态链接库给抽离出来了，我们管它叫`libWeiboSDK.i386.a`，现在我们再用`ar`命令解压它看看里面有什么

```
ar -x libWeibo.i386.a
```

解压完成后你会看到好多好多以`.o`结尾的对象文件，回忆回忆刚刚我们讲到的编译链接过程，这些对象文件就是给`链接器`最终生成静态链接库时用到的文件，由于太多了，我只列出我们要讲到的几个：

```
-rw-r--r--  1 leenjewel  staff    13K Jan  8 15:47 NSData+WBSDKBase64.o
-rw-r--r--  1 leenjewel  staff    42K Jan  8 15:47 UIImage+WBSDKResize.o
-rw-r--r--  1 leenjewel  staff    12K Jan  8 15:47 UIImage+WBSDKStretch.o
-rw-r--r--  1 leenjewel  staff    74K Jan  8 15:47 UIView+WBSDKSizes.o
-rw-r--r--  1 leenjewel  staff    58K Jan  8 15:47 WBAidManager.o
-rw-r--r--  1 leenjewel  staff    15K Jan  8 15:47 WBAuthorizeRequest.o
-rw-r--r--  1 leenjewel  staff    16K Jan  8 15:47 WBAuthorizeResponse.o
-rw-r--r--  1 leenjewel  staff    19K Jan  8 15:47 WBBaseMediaObject.o
-rw-r--r--  1 leenjewel  staff   265K Jan  8 15:47 WBSDKJSONKit.o
```

###为什么会在运行中崩溃？

当我们把新浪微博 SDK 的静态链接库引入我们自己的项目，并 Build 我们自己的项目到模拟器或真机设备上运行的过程其实也是一个编译链接的过程，最终从项目 Build 生成可以在模拟器或真机设备运行的 App，而这个过程中对新浪微博 SDK 的静态链接库的处理方式和我们刚刚拆开`libWeiboSDK.a`的过程差不多：

- 将 libWeibSDK.a 根据当前所构建的平台架构(模拟器还是真机设备)进行瘦身

- 将瘦身的静态库解压拆包

- 将用到的对象文件链接进入项目

而我们遇到的崩溃问题恰恰是出在了`将用到的对象文件链接进入项目`这一步。

苹果的开发者网站针对这个问题有一篇[说明文章](https://developer.apple.com/library/ios/qa/qa1490/_index.html)，我们来引用一下里面的内容：

>The dynamic nature of Objective-C complicates things slightly. Because the code that implements a method is not determined until the method is actually called,

这句话解释起来就是说 Objective-C 是有运行时(runtime)的，一个方法要执行什么代码是在运行时决定的，而不是在链接时决定的。想要再深入了解 Objective-C 运行时知识的，可以看看[这里](http://blog.cocoabit.com/blog/2014/10/06/yi-li-jieobjective-cruntime/)

>Objective-C does not define linker symbols for methods. Linker symbols are only defined for classes.

因为在 Objective-C 中，一个方法的执行是要到运行时才决定的，所以在链接时，链接器只链接类的符号，并不会链接方法的符号。

>For example, if main.m includes the code [[FooClass alloc] initWithBar:nil]; then main.o will contain an undefined symbol for FooClass, but no linker symbols for the -initWithBar: method will be in main.o

最后还举了一个例子：当你在`main.m`文件中初始化一个类`FooClass`的对象，然后调用了这个类`FooClass`的一个对象方法`initWithBar`，在链接器分析由`main.m`编译生成的`main.o`对象文件时，发现这个对象文件没有定义符号`FooClass`于是就会去其他`.o`对象文件中去寻找`FooClass`符号的定义，而至于方法符号`initWithBar`的定义在哪里链接器是不关心的，因为`initWithBar`的执行是由运行时负责的，链接器不管。

好了，现在问题来了，我们再重复一下这句话：

```
Objective-C 中方法的执行实在运行时决定的，所以链接器只链接类的符号，不链接方法的符号
```

我们再回过头看看崩溃的报错信息：

```
[__NSDictionaryM weibosdk_WBSDKJSONString] : unrecognized selector sent to instance 0x170255780
```

这说明崩溃的原因是在运行时调用`__NSDictionaryM`类对象的`weibosdk_WBSDKJSONString`方法时没有找到该方法的定义。这里不难看出`__NSDictionaryM`是`Foundation Framework`中的类，而方法`weibosdk_WBSDKJSONString`是新浪微博 SDK 自己定义的方法，新浪在这里使用了**分类技术**扩展了`__NSDictionaryM`类的行为。我们来验证这一点：

我们已经解压出`libWeiboSDK.a`中的全部`.o`对象文件，我们用`nm`命令导出全部对象文件中的符号：

```
nm *.o >> libWeiboSDK.symbols.txt
```

然后我们用个文本编辑器打开`libWeiboSDK.symbols.txt`查找`weibosdk_WBSDKJSONString`，我们可以查到如下结果：

```
WBSDKJSONKit.o:
00007ba0 t -[NSArray(WBSDKJSONKitSerializing) weibosdk_WBSDKJSONString]
00007de8 t -[NSDictionary(WBSDKJSONKitSerializing) weibosdk_WBSDKJSONString]
000079cd t -[NSString(WBSDKJSONKitSerializing) weibosdk_WBSDKJSONString]
```

这就可以说明新浪微博 SDK 确实使用了**分类技术**扩展了`NSArray`、`NSDictionary`和`NSString`三个 Foundation Framework 下面的类的行为。好，现在可以真相大白了：

- 在链接时，链接器发现`WBSDKJSONKit.o`对象文件中缺少类符号`NSArray`、`NSDictionary`和`NSString`。

- 链接器从`Foundation Framework`中找到了类的符号定义，从而将`Foundation Framework`中相关的对象文件链接进来

- 由于链接器不链接方法符号，所以`weibosdk_WBSDKJSONString`这样的方法符号完全被忽略了。

- 由于类符号的定义在`Foundation Farmework`中定义，所以`WBSDKJSONKit.o`对象文件中没有符号被引用，链接器就没有把这个对象文件链接进来。

- 运行时运行到`weibosdk_WBSDKJSONString`方法时，由于`Foundation Framework`中是不存在这个方法的定义的，而存在这个方法定义的`WBSDKJSONKit.o`对象文件又没有被链接器链接进来，所以崩溃了。

###为什么增加编译选项可以解决问题？

我们继续引用苹果的开发者网站针对这个问题的[说明文章](https://developer.apple.com/library/ios/qa/qa1490/_index.html)中的内容：

>Passing the -ObjC option to the linker causes it to load all members of static libraries that implement any Objective-C class or category. This will pickup any category method implementations. But it can make the resulting executable larger, and may pickup unnecessary objects. For this reason it is not on by default.

加了`-ObjC`选项后，不管是否被引用到，链接器会把 Objective-C 的类和分类的所有对象文件全部链接，全部链接后方法符号全部被链接进来，崩溃的问题自然被解决了。

而`-all_load`选项更彻底，这个选项会让链接器把全部的对象文件都链接进来，当然，代价就是构建的 APP 体积会变大。

###为什么 cocos2d-x 加了编译选项会无法编译通过？

其实准确的说法是编译可以成功进行，链接器执行报错。我们再回顾一下加了`-ObjC`或`-all_load`链接选项后链接器的报错信息：

```
Undefined symbols for architecture i386:
  "_GCControllerDidConnectNotification", referenced from:
      -[GCControllerConnectionEventHandler observerConnection:disconnection:] in libcocos2dx iOS.a(CCController-iOS.o)
  "_GCControllerDidDisconnectNotification", referenced from:
      -[GCControllerConnectionEventHandler observerConnection:disconnection:] in libcocos2dx iOS.a(CCController-iOS.o)
  "_OBJC_CLASS_$_GCController", referenced from:
      objc-class-ref in libcocos2dx iOS.a(CCController-iOS.o)
     (maybe you meant: _OBJC_CLASS_$_GCControllerConnectionEventHandler)
```

根据报错信息我们能够了解到报错是一个名叫`CCController-iOS.o`对象文件导致的，而这个文件对应的源代码是`CCController-iOS.mm`，通过阅读源码我们发现，这个文件中定义了一个 Objective-C 的类`GCControllerConnectionEventHandler`，这个类中的方法引用了`GCControllerDidConnectNotification`和`GCControllerDidDisconnectNotification`两个类，而这两个类实在`GameController Framework`中定义的。

而 cocos2d-x 生成的项目默认并没有为我们引入`GameController Framework`，所以在链接时由于链接器找不到对应类的符号定义，所以才会报错。如果你到 Xcode->Target->Buid   Phases-> 下   Link Binary With Libraries   项添加`GameController Framework`就可以解决问题了，但是这种解决方式**很不干净**

###正确的姿势

`-force_load path/to/your/libWeiboSDK.a`链接选项其实是干了和`-ObjC`、`-all_load`一样的事情，只不过它更有针对性，它只让链接器把你指定的静态链接库中的全部对象文件链接进来，这样更清爽一些。

希望我的解释已经够深入了。

:-)





