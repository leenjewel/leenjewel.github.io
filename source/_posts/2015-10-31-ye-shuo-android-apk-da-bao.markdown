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

###观测结果：

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

###观测结果：

- 1、改变资源路径顺序后 R.java 文件中资源 ID 的值发生了变化。