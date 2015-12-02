---
layout: post
title: "也说 Android apk 打包"
date: 2015-12-02 18:29:48 +0800
comments: true
categories: android
---

我们在开发 Android 应用程序的时候一般都只需要使用 Eclipse 或是 Android Studio 这样的 IDE 编写好业务逻辑，最终由这些开发工具来协助我们将代码打包生成最终可以在设备上运行的 APK 包。正因为有这些集成度很高的开发工具，我们很少能够接触到 Android APK 包生成的具体流程。

其实如果你 Google 一下“Android apk 打包流程”或者“手动打包 Android apk”，能找到很多介绍 Android APK 打包流程的内容，而我为什么又要再多写一篇呢？是因为这些内容大多只介绍了一个标准的 Android 工程是如何一步步变成 apk 包的，而很少有写一个标准 Android 工程附带依赖几个 Android Library 的情况又是怎样的。要知道在国内的 Android 开发环境下你的产品不接入几个第三方 SDK 你都不好意思出门跟人家到招呼！而往往一个产品根据发放的渠道不同，又要接入不同的第三方 SDK。所以我想从这些方面入手来写这一篇。另外，网上大多数文章都已经有些过时了，例如大多数最后都提到用 `apkbuilder` 这个脚本来生成最终的 apk 包，而实际上目前最新的 Android SDK 早就已经移除了这个脚本，更改了最终生成 apk 包的方式。

##概述

一个 Android 工程最后变成 apk 包大概要做这么几件事儿：

- 1、生成 `R.java` 文件
- 2、将 `.java` 文件编译成 `.class` 文件
- 3、将 `.class` 文件打包成 `.jar` 文件
- 4、将所有 `.jar` 文件（包括依赖库）编译成 `classes.dex` 文件
- 5、将 `assets` 和 `res` 文件夹中所有的资源文件打包成一个 `apk` 包
- 6、将 `classes.dex` 文件添加进 `apk` 包
- 7、如果有使用 `NDK` 技术的话，将生成的 `.so` 文件添加进 `apk` 包
- 8、对 `apk` 包进行签名

说白了一个 apk 包就是由**代码**和**资源**组成的。代码的处理基本上就是编译，这个没什么可说的。我们主要说说资源打包。我们通过亲手实践来理解 apk 生成中的资源处理过程。

##建立试验环境

找一个空白目录，建立一个标准的 Android 工程，再建立两个标准的 Android Library 工程。

```sh
$android create project --activity MainActivity --package com.leenjewel.test --path ./AndroidTestProject -t android-21

$android create lib-project --package com.leenjewel.test.liba --path ./AndroidLibProjectA -t android-21

$android create lib-project --package com.leenjewel.test.libb --path ./AndroidLibProjectB -t android-21
```

怎么样？是不是有些同学连命令行手工建立 Android 工程都是头一次看到啊？可以去刚刚新建好的三个工程的目录看看，其实 Android 的标准工程和 Library 工程并没有什么太大的区别。

##【试验1】第一次生成 R.java 文件

然后我们切换到刚刚建立的标准 Android 工程的目录下面，准备手工生成 `R.java` 文件。`R.java` 文件是什么我就不解释了，这里我们根据主项目和两个 Libary 项目分别生成三个 `R.java` 文件，后面我们再做解释，先一步一步跟着做。

```sh
$aapt package -m -J ./gen -M ./AndroidManifest.xml \
    -S ./res \
    -S ../AndroidLibProjectA/res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    --auto-add-overlay

$aapt package -m -J ./gen -M ../AndroidLibProjectA/AndroidManifest.xml  \
    -S ./res \
    -S ../AndroidLibProjectA/res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    --auto-add-overlay \
    --non-constant-id

$aapt package -m -J ./gen -M ../AndroidLibProjectB/AndroidManifest.xml \
    -S ./res  \
    -S ../AndroidLibProjectA/res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    --auto-add-overlay \
    --non-constant-id
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

每一个工程的 `res/drawable-*/` 目录下面都有一个自动生成的 `ic_launcher.png` 文件，他们的内容都是一样的：“经典的 Android 机器人”。为了区分他们，我们随便用个图片编辑工具在两个 Library 工程的 `ic_launcher.png` 的小机器人胸前按照它们所属的项目分别打上 `A` 和 `B` 两个标签。

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
$aapt package -m -J ./gen -M ./AndroidManifest.xml \
    -S ./res \
    -S ../AndroidLibProjectA/res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    --auto-add-overlay
```

现在我们调整一下这个命令里 `-S` 参数的顺序，将 `AndroidLibProjectA/res` 放在第一，其他不变：

```sh
$aapt package -m -J ./gen -M ./AndroidManifest.xml \
    -S ../AndroidLibProjectA/res \
    -S ./res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    --auto-add-overlay
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
$aapt package -f -M AndroidManifest.xml \
    -S ./res \
    -S ../AndroidLibProjectA/res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    -F ./out/res.apk \
    --auto-add-overlay
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
$aapt package -f -M AndroidManifest.xml \
    -S ./res \
    -S ../AndroidLibProjectA/res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    -F ./out/res.apk \
    --auto-add-overlay
```

然后我们依然是调整一下 `-S` 命令参数的顺序和刚才一样我们把主工程的 `res` 目录和依赖库工程 AndroidLibProjectA 的 `res` 目录调换顺序再生成一下 apk 包看看：

```sh
$aapt package -f -M AndroidManifest.xml \
    -S ../AndroidLibProjectA/res \
    -S ./res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    -F ./out/res.apk \
    --auto-add-overlay
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

细心的同学相比已经有结论了。如果单单只是一个 Android 标准工程的资源打包的话那就轻松很多了，直接一个命令就搞定。而附带诸多 Library 工程依赖的话就要考虑资源的合并问题了，而资源合并过程中主要要处理的就是同名资源问题。通过我们做过的试验我们可以得出一个结论即：

> **Android 的资源打包过程中对于资源特别是同名资源的处理只和资源路径的先后顺序有关系，只要资源路径顺序不变，在 R.java 文件中生成的资源引用 ID 就不变**。

资源打包的事情就说到这里。顺带提一句，上面的试验中并没有出现 `assets` 资源文件打包的情况，其实 `assets` 资源文件打包和 `res` 资源文件打包的原理和规则都是一样的，加上 `assets` 资源打包的命令看起来大概是这个样子的

```sh
$aapt package -f -M AndroidManifest.xml \
    -A ./assets \
    -A ../AndroidLibProjectA/assets \
    -A ../AndroidLibProjectB/assets \
    -S ./res \
    -S ../AndroidLibProjectA/res \
    -S ../AndroidLibProjectB/res \
    -I ~/Dev/android-sdk-macosx/platforms/android-21/android.jar\
    -F ./out/res.apk \
    --auto-add-overlay
```

`assets` 资源文件对于同名文件资源的合并处理规则也是和资源路径顺序有关。

##编译源代码

- 编译 java

```sh
javac -target 1.7  -source 1.7 \
    -d  ./bin \
    -bootclasspath  ~/Dev/android-sdk-macosx/platforms/android-21/android.jar \
    -classpath ./libs/lib1.jar:./libs/lib2.jar:./libs/lib3.jar \
    ./src/com/package/your/xxxx.java \
    ./src/com/package/your/xxxxx.java \
    ./src/com/package/your/xxxxxx.java
    ....
```

这里要说的是 Android Library 工程和 Android 主工程的源代码是分开编译的。因为主工程中的代码是依赖 Library 工程的，所以要将所有 Library 工程编译并生成 `jar` 文件，然后再编译主工程时将所有 Library 工程的 `jar` 引入 `classpath` 中。

主工程编译的时候记得要连同之前生成在 `gen` 文件夹中的 `R.java` 文件和 `src` 文件夹下面的所有源代码一起编译。

- 生成 jar

```sh
jar  cvf  ./bin/classes.jar \
    -C ./bin  .
```

- 生成 dex

```sh
dx  --dex \
    --output  ./bin/classes.dex \
    ./bin/classes.jar \
    ./libs/lib1.jar \
    ./libs/lib2.jar \
    ./libs/lib3.jar \
    ../AndroidLibProjectA/libs/lib1.jar \
    ../AndroidLibProjectA/libs/lib2.jar \
    ../AndroidLibProjectB/libs/lib1.jar \
    ../AndroidLibProjectB/libs/lib2.jar
    ......
```

这样所有的 Java 代码和依赖库就全部被编译成最终 Android apk 中要用到的 `classes.dex` 文件了。

##生成最终的 apk 包

- 添加 classes.dex 文件

然后我们把生成的 `classes.dex` 文件丢进刚刚生成的资源 apk 包里。原本这步是通过 `apkbuilder` 脚本来做的，现在 Google 统一改成用 `aapt` 命令来做了。

```sh
cd ./bin
aapt add ../out/res.apk  classes.dex
```

这里需要注意的是 `classes.dex` 文件前面一定不要加额外的路径，如果加了路径那么这些路径会一并带进 apk 包里。而 Android apk 包里的 `classes.dex` 文件是直接放进包里面的，不能带路径。

- 添加 .so 文件

如果你使用了 NDK 技术或者有库依赖，那也需要将这些依赖的 `.so` 文件添加进 apk 包，方法和添加 `classes.dex` 一样。

```sh
$aapt add res.apk lib/armeabi/xxx.so
```

据说 Windows 环境下执行上述 `aapt add` 命令时路径也要用类 Unix 系统的斜杠而不能用 Windows 的反斜杠，我手边没有 Windows 环境没有亲自尝试。

- 签名

```sh
jarsigner  -digestalg  SHA1 \
    -sigalg  MD5withRSA \
    -verbose \
    -keystore  /your/keystore/file/path/xxxx.keystore \
    -storepass  xxxxxxxx \
    -keypass  xxxxxxxx \
    -signedjar  ./out/res.signed.apk \
    ./out/res.apk  "your keystore alias"
```

- 优化 apk 包

这是 Google 官方文档提到的将 apk 包文件做一下对齐操作。

```
zipalign  -f  4  ./out/res.signed.apk  ./out/your.final.apk
```

至此一个可放在 Android 设备上运行的 apk 包就成功的生成了。

##Android 第三方库的集成方式

前面也说了在国内特殊的 Android 开发环境下你的 Android 应用不接入几个第三方 SDK 你出门都不好意思和别人打招呼。甚至有时我们为了针对各种不同的渠道，一个 Android 应用程序出包时要不停的切换各种第三方 SDK 依赖，想必搞过 Android 开发的人都知道这是个很头疼的问题。

有了前面针对 Android apk 打包流程的介绍，我们最后再来探讨一下 Android 出包，特别是接入各种第三方 SDK ，针对不同渠道切换不同依赖出包的更为灵活的方式的思路。

###【集成方式1】以标准 Android Library Project 方式集成

我们一般在使用自己内部开发的一些公共库或公共组件时会选用这种方式。所有的资源和代码都以标准的 Android Library Porject 的形式呈现，开发应用时只需要声明引用这些库工程即可。

通过刚刚的试验我们知道只要引入资源的路径顺序不变，主工程和依赖库工程所生成的 `R.java` 文件中资源 ID 的值是一样的。所以在开发依赖库工程时，在依赖库工程中通过自己的 R 类来引用资源写好的逻辑代码在主工程中同样是可以正常工作的。

###【集成方式2】依赖库通过复制粘贴进主工程的方式集成

如果我们开发的库要发布给第三方使用的时候，一般会将代码混淆打包成 `jar` 文件连同资源文件一起发送给第三方。第三方拿到这样形式的 SDK 后一般会通过复制粘贴的方式将资源文件拷贝进主工程的 `res` 或 `assets` 文件夹内，将 `jar` 包拷贝进主工程的 `libs` 文件夹内。

这时由于库使用的资源文件是直接放置在主工程中的，所以最终生成的资源 ID 是不同的。存在于第三方库 `jar` 包中的库逻辑如果还是按照 R 类来引用资源的话就不行了。因为库逻辑是不可能预先知道主工程的包名的，这时只能通过 Java 的“反射”机制来进行资源引用了。举个例子：

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

###【集成方式3】通过二次打包的方式集成

其实说到针对不同渠道不同第三方 SDK 依赖出包的问题，很多同学首先想到的就是使用如  [AnySDK](http://www.anysdk.com) 这样的一整套解决方案。使用这种方案你只需要在你的主项目中薄薄的接入一层 AnySDK 的中间层 SDK 依赖，然后将你生成的 apk 包交给 AnySDK 的客户端，通过 UI 界面选择好你需要的第三方 SDK 依赖，然后等待进度条走完之后一个复合你要求的渠道包就生成出来了。

可是在你享受这种方便的时候是否思考过它是**如何实现**的呢？

当你依赖的第三方 SDK 并不在 AnySDK 这种解决方案提供商的支持之列，而支持计划遥遥无期的时候，当你生成的渠道包发生问题去寻求 AnySDK 的反馈而没有响应时，你是否咬牙切齿的想要自己也搞这么一套方便的解决方案把一切**掌控在自己手中**呢？

这里就告诉你它们所使用的是什么样的**黑科技**。

其实了解了 Android apk 的打包原理后，只要稍稍想一想就会明白所谓的黑科技也并不是什么复杂的东西，无非是：

- 建立统一的接口规范，将各个第三方 SDK 的调用方式统一化、规范化。

- 将各个第三方 SDK 按照统一接口规范做一次二次封装，使其暴露的对外调用接口统一。

- 只将薄薄的一层统一调用接入标准工程项目中，排除了各种第三方依赖。

- 通过二次打包的方式将所需的各个第三方 SDK 打进 apk 包。

- 最后运行时通过动态加载技术经过统一的接口调用各个第三方 SDK。

这里的重点在于二次打包。当然不排除各家类 AnySDK 解决方案提供商有各家的方式方法，不过大概的原理是相通的。

- 用类似 [apktool](https://ibotpeaches.github.io/Apktool/) 这样的工具将原 apk 包解包。

- 通过 `aapk` 工具将各个第三方 SDK 中的资源合并，重新生成 R.java 并编译成 R.class 文件替换原包中的 R.class 文件，然后根据各个第三方 SDK 的包路径不同对应生成各个包下的 R.java 文件。

- 将新生成的各个 class 文件和各个第三方 SDK 中的依赖 jar 包和原 apk 包中的 class 文件一起重新编译成 `classes.dex` 文件。

- 最后重新打包成新的 apk 然后重新签名一次即可。

##最后的广告时间

[MySDK](https://github.com/leenjewel/mysdk) 项目就是我仿照 AnySDK 这样的解决方案实现的一套框架。虽然还没有完工，但基本该有的东西都有了：

- 统一接口的中间层 C++、Java、Objective-C、Cocos2d-x Lua 语言层面的调用支持。

- 基于 Web 和基于命令行的二次打包工具

有兴趣的同学可以 clone 下来玩玩。附上简单玩法：

提前部署好你的 Android 环境。Android SDK 一定要有，并且要定义一个环境变量 `$ANDROID_SDK_ROOT` 指明 Android SDK 的位置。JDK 要安装这个更不用说了，还有 Python 环境。

Web 二次打包工具基于 Python 的 [Tornado Web Framework](http://www.tornadoweb.org) 所以请先自行安装。然后：

```
$git clone git@github.com:leenjewel/mysdk.git

$cd mysdk

$cd apk.builder

$python setup.py install

$python mysdk.py start-server \
    --server-config ./../example/apk_build_test/mysdk_web_server/settings.py \
    --with-port 8080

```

如果命令执行成功的话，直接打开浏览器访问 [http://localhost:8080/index](http://localhost:8080/index) 就可以看到二次打包工具的 Web UI 界面了。

先新建一个工作空间(workspace)，点击 `New Workspace` 按钮新建好工作空间后继续点击 `Go To Workspace` 按钮进入新建好的工作空间。

在新建好的工作空间中点击 `New Project` 按钮新建一个二次打包项目。这里需要注意的是：

- APK File 文件选择 `example/apk_build_test/MySDKAPPExample.apk`

- APK Package Name 填写 `com.leenjewel.mysdk.appexample`

- Keystore File 文件选择 `example/android/MySDKAPPExample/keystore` 文件

- Store Password 和 Key Password 都填写 `com.leenjewel.mysdk`

- Alias 填写 mysdk

- `aexamplesdk` 点击 `Add SDK` 按钮添加，`metadata_val_1` 的值随便填写，这表示二次打包将这个名叫 `aexamplesdk` 的第三方 SDK 加入到原包中。

接着点击 `Create Project` 按钮完成二次打包项目的创建。

最后点击 `Build Project` 按钮进入到二次打包界面，再次确认项目配置填写无误的话直接点击二次打包界面里的 `Build Project` 按钮就开始二次打包操作了。

祝玩的愉快。如遇到问题欢迎和我讨论。