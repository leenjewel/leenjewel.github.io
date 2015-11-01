---
layout: post
title: "也说 Android apk 打包"
date: 2015-10-31 21:52:43 +0800
comments: true
categories: android
---

我们在开发 Android 应用程序的时候一般都只需要使用 Eclipse 或是 Android Studio 这样的 IDE 编写好业务逻辑，最终由这些开发工具来协助我们将代码打包生成最终可以在设备上运行的 APK 包。正因为有这些集成度很高的开发工具，我们很少能够接触到 Android APK 包生成的具体流程。

这里先多罗嗦几句。我一直认为想要彻底掌握一门技术就一定要深入地去了解它底层的一些东西，要明白它的原理才行。随着我们对工作效率的追求，我们使用的各种开发工具的自动化程度越来越高，这也使得我们在享受到这些工具给我们带来的便利的同时也让我们越来越难以接触到很多底层触及到原理的东西。举个最简单的例子：我曾经面试过一个刚刚大学毕业的小伙子，问他：“Python 字符串的分割方法是什么？”，他想了半天支支吾吾的回答：“呃~有些忘记了，反正是有这么个方法”。其实我自己也遇到过这种情况，用带语法提示的 IDE 多了就会这样。

好了，废话就说到这。其实如果你 Google 一下“Android apk 打包流程”或者“手动打包 Android apk”，能找到很多介绍 Android APK 打包流程的内容，而我为什么又要再多写一篇呢？是因为这些内容大多只介绍了一个标准的 Android 工程是如何一步步变成 apk 包的，而很少有写一个标准 Android 工程附带依赖几个 Android Library 的情况又是怎样的。要知道在国内的 Android 开发环境下你的产品不接入几个第三方 SDK 你都不好意思出门跟人家到招呼！而往往一个产品根据发放的渠道不同，又要接入不同的第三方 SDK。所以我想从这些方面入手来写这一篇。

首先我们先来做几个试验。

##建立试验环境

找一个空白目录，建立一个标准的 Android 工程，再建立两个标准的 Android Library 工程。

```sh
$android create project --activity MainActivity --package com.leenjewel.test --path ./AndroidTestProject -t android-21

$android create lib-project --package com.leenjewel.test.liba --path ./AndroidLibProjectA -t android-21

$android create lib-project --package com.leenjewel.test.libb --path ./AndroidLibProjectB -t android-21
```

怎么样？是不是有些同学连命令行手工建立 Android 工程都是头一次看到啊？

##【试验1】第一次生成 R.java 文件

然后我们切换到刚刚建立的标准 Android 工程的目录下面，准备手工生成 `R.java` 文件。`R.java` 文件是什么我就不解释了，这里我们根据主项目和两个 Libary 项目分别生成三个 `R.java` 文件，后面我们再做解释，先一步一步跟着做。

```sh
$aapt package -m -J ./gen -M ./AndroidManifest.xml -S ./res -S ../AndroidLibProjectA/res -S ../AndroidLibProjectB/res -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar --auto-add-overlay

$aapt package -m -J ./gen -M ../AndroidLibProjectA/AndroidManifest.xml  -S ./res -S ../AndroidLibProjectA/res -S ../AndroidLibProjectB/res -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar --auto-add-overlay --non-constant-id

$aapt package -m -J ./gen -M ../AndroidLibProjectB/AndroidManifest.xml -S ./res  -S ../AndroidLibProjectA/res -S ../AndroidLibProjectB/res -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar --auto-add-overlay --non-constant-id
```

如果三个命令都执行成功的话会在刚刚建立的标准 Android 工程根目录的 `gen` 目录下根据不同的子目录产生三个 R.java 文件： 

- `./gen/com/leenjewel/test/R.java`

```java

package com.leenjewel.test;

public final class R {
    public static final class attr {
    }
    public static final class drawable {
        public static final int ic_launcher=0x7f020000;
    }
    public static final class layout {
        public static final int main=0x7f030000;
    }
    public static final class string {
        public static final int app_name=0x7f040000;
    }
}
```

-  `./gen/com/leenjewel/test/liba/R.java`

```java

package com.leenjewel.test.liba;

public final class R {
    public static final class attr {
    }
    public static final class drawable {
        public static int ic_launcher=0x7f020000;
    }
    public static final class layout {
        public static int main=0x7f030000;
    }
    public static final class string {
        public static int app_name=0x7f040000;
    }
}
```

-  `./gen/com/leenjewel/test/libb/R.java`

```java

package com.leenjewel.test.libb;

public final class R {
    public static final class attr {
    }
    public static final class drawable {
        public static int ic_launcher=0x7f020000;
    }
    public static final class layout {
        public static int main=0x7f030000;
    }
    public static final class string {
        public static int app_name=0x7f040000;
    }
}
```

###【试验1】观测结果：

- 1、三个 R.java 文件中资源的 ID 相同。
- 2、三个 R.java 文件的 package 不同。
- 3、Library 工程生成的 R.java 文件的资源属性没有 final 标识。

第一个试验我们得出了三个观测结果，我们的试验继续。通过刚刚生成的三个 R.java 文件不难看出现在这三个工程各自持有的资源是一样的，接下来我们先要给这三个工程加点儿不一样的东西进去。

###特别提示：

接下来为了更好的观察试验结果，我们可以用熟悉的代码管理工具如 git 或 svn 给主工程 `AndroidTestProject` 初始化一个代码版本库并进行一次提交，这样通过 diff 我们可以更方便的观察试验变化。

每一个工程的 `res/drawable-*/` 目录下面都有一个自动生成的 `ic_launcher.png` 文件，他们的内容都是一样的：“经典的 Android 机器人”。为了区分他们，我们在两个 Library 工程的 `ic_launcher.png` 的小机器人胸前按照它们所属的项目分别打上 `A` 和 `B` 两个标签。

然后再分别编辑 `res/values/strings.xml` 资源文件，加入一些新的字符串节点。

- `AndroidTestProject/res/values/strings.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">MainActivity</string>
    <!-- new!!! -->
    <string name="same_thing">SameP</string>
    <string name="p_thing">PThing</string>
</resources>
```

- `AndroidLibProjectA/res/vaues/strings.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">A</string>
    <!-- new!!! -->
    <string name="same_thing">SameA</string>
    <string name="a_thing">AThing</string>
</resources>
```

- `AndroidLibProjectB/res/values/strings.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">B</string>
    <!-- new!!! -->
    <string name="same_thing">SameB</string>
    <string name="b_thing">BThing</string>
</resources>
```

做完了这些改变后如果已经给主工程建立了代码库的同学这个时候可以再次提交一下。然后我们准备进行下一个试验。

我们先再次重新生成一下前面刚刚生成过的三个 `R.java` 文件。生成后可以发现新增加了几个资源 ID，但是我们刚刚观测的结果没变，生成的三个 `R.java` 文件的资源 ID 依然是一致的。出于篇幅考虑这里就不再放出三个 R.java 文件的内容了，只放了主工程的 R.java 文件内容。

- `AndroidTestProject/gen/com/leenjewel/test/R.java`

```java

package com.leenjewel.test;

public final class R {
    public static final class attr {
    }
    public static final class drawable {
        public static final int ic_launcher=0x7f020000;
    }
    public static final class layout {
        public static final int main=0x7f030000;
    }
    public static final class string {
        public static final int a_thing=0x7f040003;
        public static final int app_name=0x7f040000;
        public static final int b_thing=0x7f040002;
        public static final int p_thing=0x7f040004;
        public static final int same_thing=0x7f040001;
    }
}
```

这里我们修改过资源后重新生成三个 `R.java` 文件，再次提交，准备新的试验。

##【试验2】改变资源路径顺序重新生成 R.java 文件

以主工程为例，我们刚刚一直使用的生成主工程 R.java 文件的命令是：

```sh
$aapt package -m -J ./gen -M ./AndroidManifest.xml -S ./res -S ../AndroidLibProjectA/res -S ../AndroidLibProjectB/res -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar --auto-add-overlay
```

现在我们调整一下这个命令里 `-S` 参数的顺序，将 `AndroidLibProjectA/res` 放在第一，其他不变：

```sh
$aapt package -m -J ./gen -M ./AndroidManifest.xml -S ../AndroidLibProjectA/res -S ./res -S ../AndroidLibProjectB/res -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar --auto-add-overlay
```

这时再次生成主工程的 `R.java` 文件看看有什么变化：

- `AndroidTestProject/gen/com/leenjewel/test/R.java`

```java

package com.leenjewel.test;

public final class R {
    public static final class attr {
    }
    public static final class drawable {
        public static final int ic_launcher=0x7f020000;
    }
    public static final class layout {
        public static final int main=0x7f030000;
    }
    public static final class string {
        public static final int a_thing=0x7f040004;
        public static final int app_name=0x7f040000;
        public static final int b_thing=0x7f040002;
        public static final int p_thing=0x7f040003;
        public static final int same_thing=0x7f040001;
    }
}
```

###【试验2】观测结果：

- 1、改变资源路径顺序后 R.java 文件中资源 ID 的值发生了变化。

注意，这次我们不提交文件的变动，将主工程还原到调整资源路径顺序之前的状态然后继续我们的试验。

##【试验3】将资源生成 APK 包

下面我们要将主工程和两个依赖库工程的资源打包，这也是 Android 应用程序 apk 包生成过程中的步骤之一。这里我们直接给出命令：

```sh
$aapt package -f -M AndroidManifest.xml -S ./res -S ../AndroidLibProjectA/res -S ../AndroidLibProjectB/res -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar -F ./out/res.apk --auto-add-overlay
```

这个命令看上去多多少少和刚刚生成 R.java 的命令有些类似，如果执行顺利的话会在主工程根目录下面的 `out` 目录下生成一个叫做 `res.apk` 的包。当然这个包并不能在 Android 设备上运行，这只是个半成品，我们继续。

了解 Android 开发的人都知道 apk 其实就是一个 zip 压缩包，用 zip 解压缩软件就可以进行解压缩操作。但是当我们尝试通过 zip 解压缩我们刚刚生成的 `res.apk` 后发现里面并不是简简单单的将我们的资源文件直接打包进去而是额外的还做了一些类似于编译的操作，例如我们之前编辑过的 `res/values/strings.xml` 资源就已经被“编译”成为 `resources.arsc` 文件的一部分，而诸如 `AndroidManifest.xml` 这样的文件也不再是明文的我们可以看懂的格式了。

这时我们需要一个工具将我们刚刚生成的这个 `res.apk` 包“反编译”回来，这个好用的工具就是 [apktool](https://ibotpeaches.github.io/Apktool/)，下载这个工具后我们来“反编译”我们的 apk 包：

```sh
$java -jar apktool.jar d res.apk
```

如果执行顺利的话我们在 `res.apk` 包所在的同级目录下可以看到一个 `res` 目录，里面就是刚刚通过 apktool 反编译回来的文件，我们重点观察两个文件：

- `ic_launcher.png` 文件
- `res/values/strings.xml` 文件

由于之前我们给主工程和两个依赖库工程的 `ic_launcher.png` 文件做过标记，所以这时候不难看出：

###【试验3】观测结果：

- 1、`res.apk` 包中 `ic_launcher.png` 文件来自 AndroidTestProject 即来自主工程。
- 2、`res.apk` 包中 `res/values/strings.xml` 文件的资源 `same_thing` 和 `app_name` 的值来自 AndroidTestProject 即来自主工程。

- `res/values/strings.xml` 文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">MainActivity</string>
    <string name="same_thing">SameP</string>
    <string name="b_thing">BThing</string>
    <string name="a_thing">AThing</string>
    <string name="p_thing">PThing</string>
</resources>
```

不用提交任何代码，我们试验继续。

##【试验4】改变资源路径顺序重新生成 APK 包

把刚刚第一次生成的 `res.apk` 和用 apktool 反编译出来的 `res` 目录都删掉。为了方便对比，再看一眼刚刚用来生成 apk 的命令：

```sh
$aapt package -f -M AndroidManifest.xml -S ./res -S ../AndroidLibProjectA/res -S ../AndroidLibProjectB/res -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar -F ./out/res.apk --auto-add-overlay
```

然后我们依然是调整一下 `-S` 命令参数的顺序和刚才一样我们把主工程的 `res` 目录和依赖库工程 AndroidLibProjectA 的 `res` 目录调换顺序再生成一下 apk 包看看：

```sh
$aapt package -f -M AndroidManifest.xml -S ../AndroidLibProjectA/res -S ./res -S ../AndroidLibProjectB/res -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar -F ./out/res.apk --auto-add-overlay
```

重新生成新的 `res.apk` 包之后我们还是用 apktool 将包进行反编译然后观察：

###【试验4】观测结果：

- 1、`res.apk` 包中 `ic_launcher.png` 文件来自 AndroidLibProjectA 即来自依赖库 A 工程。
- 2、`res.apk` 包中 `res/values/strings.xml` 文件的资源 `same_thing` 和 `app_name` 的值来自 AndroidLibProjectA 即来自依赖库 A 工程。

- `res/values/strings.xml` 文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">A</string>
    <string name="same_thing">SameA</string>
    <string name="b_thing">BThing</string>
    <string name="p_thing">PThing</string>
    <string name="a_thing">AThing</string>
</resources>
```
如果你觉得意犹未尽那么可以尝试再次调整资源路径顺序，这次把依赖库 B 工程的资源目录的顺序放在最前面然后重复试验 4 的步骤再次生成 `res.apk` 包并使用 apktool 反编译之后观察试验结果。而我们不打算再这么做了，因为结果已经很明确了。

试验就暂时做到这，接下来我们说说通过这四个无聊的小试验我们了解到了什么。

##Android APK 生成过程

Google 一下网络上详细介绍这个的内容很多，这里我们只做个概括总结：

- 生成 `R.java` 文件，这个刚刚我们做过。
- 将资源打包成 `apk` 文件，这个我们刚刚也做过。
- 将 Java 源文件编译成 class 文件。
- 通过 `dex` 命令将刚刚编译的 class 文件编译成 `dex` 文件
- 将刚刚生成的 `dex` 文件通过 `aapt add` 命令加入到刚刚生成的 `apk` 包中。
- 如果应用程序使用了 NDK ，则还要将 `libs` 目录中的 `.so` 链接库文件也通过 `aapt add` 命令加入到刚刚生成的 `apk` 包中。
- 最后对 `apk` 包进行签名。大功告成~

这里需要多唠叨几句的是：网络上大多告诉最后通过一个叫做 `apkbuilder` 的工具生成最终的 `apk` 包的方式已经过时了。Google 已经在 Android SDK 中将这个工具给删除掉了，现在都是通过 `aapt add` 命令来往资源包里添加其他文件的方式来形成最终的 `apk` 包了。

放置在工程 `libs` 目录下的各种链接库文件也要先集中放置在一个 `lib` 目录下面然后在统一通过 `aapt add` 命令进行引入操作。例如：

```sh
$aapt add res.apk lib/armeabi/xxx.so
```

据说 Windows 环境下执行上述 `aapt add` 命令时路径也要用类 Unix 系统的斜杠而不能用 Windows 的反斜杠，我手边没有 Windows 环境没有亲自尝试。

前面也说了在国内特殊的 Android 开发环境下你的 Android 应用不接入几个第三方 SDK 你出门都不好意思和别人打招呼。甚至有时我们为了针对各种不同的渠道，一个 Android 应用程序出包时要不停的切换各种第三方 SDK 依赖，想必搞过 Android 开发的人都知道这是个很头疼的问题。

而我写这篇主要就是要探讨这个问题的解决办法，包括前面做的这四个试验和 Android apk 打包原理的介绍都是为了接下来我们要探讨的东西做个铺垫，只有充分了解了上面这些基本原理才能找到 Android 应用出包时如何解决复杂的第三方 SDK 依赖问题的有效方法。

结合我们试验了解到的知识从简到难先来说说三种 Android 集成第三方依赖的方式。

##【集成方式1】以标准 Android Library Project 方式集成

我们一般在使用自己内部开发的一些公共库或公共组件时会选用这种方式。所有的资源和代码都以标准的 Android Library Porject 的形式呈现，开发应用时只需要声明引用这些库工程即可。

通过刚刚的试验我们知道只要引入资源的路径顺序不变，主工程和依赖库工程所生成的 `R.java` 文件中资源 ID 的值是一样的。所以在开发依赖库工程时，在依赖库工程中通过自己的 R 类来引用资源写好的逻辑代码在主工程中同样是可以正常工作的。

##【集成方式2】依赖库通过复制粘贴进主工程的方式集成

如果我们开发的库要发布给第三方使用的时候，一般会将代码混淆打包成 `jar` 文件连同资源文件一起发送给第三方。第三方拿到这样形式的 SDK 后一般会将资源文件合并进主工程的 `res` 文件夹内，将 `jar` 包合并进主工程的 `libs` 文件夹内。

这时由于库使用的资源文件是直接放置在主工程中的，所以最终生成的资源 ID 是不同的。存在于 `jar` 包中的库逻辑如果还是按照 R 类来引用资源的话就不行了。因为库逻辑是不可能预先知道主工程的包名的，这时只能通过 Java 的“反射”机制来进行资源引用了。举个例子：

```java
private int getResId(String resType, String resName) {
	try {
		Class localClass = Class.forName(getPackageName() + ".R$" + resType);
		Field localField = localClass.getField(resName);
		return Integer.parseInt(localField.get(localField.getName()).toString());
	} catch (ClassNotFoundException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	} catch (NoSuchFieldException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	} catch (NumberFormatException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	} catch (IllegalAccessException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	} catch (IllegalArgumentException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}
	return 0;
}
```

这是个通过 Java “反射”机制查找资源 ID 的方法。一般我们引用资源时会这样写：

```java
Button btn = (Button)findViewById(R.id.btn1);
```

而在这种方式引入的依赖库中一般会通过上述的自有方法来获取资源 ID

```java
Button btn = (Button)findViewById(getResId("id", "btn1"));
```

##【集成方式3】通过二次打包的方式集成