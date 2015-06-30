---
layout: post
title: "在 Cocos2d-x 中使用 OpenSSL"
date: 2015-06-30 14:13:22 +0800
comments: true
categories: cocos2d
---

在我们使用 Cocos2d-x 引擎制作游戏过程中经常会遇到诸如对数据进行加密、解密、MD5、SHA1 散列计算等操作的需求。对于这样的需求使用 OpenSSL 库来解决是最为方便的。下面我们就说说如何将 OpenSSL 库集成到 Cocos2d-x 项目中并在 iOS 和 Android 平台下使用。什么？！不知道 OpenSSL 是什么？这么大名鼎鼎的开源项目，[移步 Wiki ](https://zh.wikipedia.org/wiki/OpenSSL)去了解吧。

##下载 OpenSSL 源代码

直接到 OpenSSL 开源项目官网去[下载](https://www.openssl.org/source/openssl-1.0.2c.tar.gz)最新版本即可。我下载的是`openssl-1.0.2c`

##编译生成 iOS 平台下适用的 OpenSSL 静态链接库

首先解压缩你下载的 OpenSSL 源代码压缩包。

```sh
tar -zxvf openssl-1.0.2c.tar.gz
```

玩过 Linux 的人都知道从源代码编译程序一般需要三步：

1. ./Configure
2. make
3. make install

编译 OpenSSL 生成静态链接库的过程也一样，唯一的区别是我们要针对不同的平台架构生成针对每个平台架构的静态链接库。例如 iPhone、iPad 目前有三种架构：`arm64`、`armv7s`和`armv7`外加模拟器的架构 `i386`，所以我们要重复上面三个步骤 4 次，生成四个平台架构对应的静态链接库。

这里我们偷个懒，用[ GitHub 上 Raphaelios 写的 Shell 脚本](https://raw.githubusercontent.com/Raphaelios/raphaelios-scripts/master/openssl/build-openssl.sh)来直接编译 OpenSSL ，下面我贴出脚本的源码并简单添加一些说明注释。

```sh
#!/bin/bash
#
#  Copyright (c) 2013 Claudiu-Vlad Ursache <claudiu@cvursache.com>
#  MIT License (see LICENSE.md file)
#
#  Based on work by Felix Schulze:
#
#  Automatic build script for libssl and libcrypto 
#  for iPhoneOS and iPhoneSimulator
#
#  Created by Felix Schulze on 16.12.10.
#  Copyright 2010 Felix Schulze. All rights reserved.
#
#  Licensed under the Apache License, Version 2.0 (the "License");
#  you may not use this file except in compliance with the License.
#  You may obtain a copy of the License at
#
#  http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
#  limitations under the License.

# 当执行时使用到未定义过的变量，则显示错误信息
set -u
 
# Setup architectures, library name and other vars + cleanup from previous runs

# 四个平台架构标识
ARCHS=("arm64" "armv7s" "armv7" "i386")

# 四个平台架构分别对应的 SDK 名称
SDKS=("iphoneos" "iphoneos" "iphoneos" "macosx")

# 使用的 OpenSSL 库版本
LIB_NAME="openssl-1.0.2c"

# 临时输出目录
TEMP_LIB_PATH="/tmp/${LIB_NAME}"
LIB_DEST_DIR="lib"
HEADER_DEST_DIR="include"
rm -rf "${HEADER_DEST_DIR}" "${LIB_DEST_DIR}" "${TEMP_LIB_PATH}*" "${LIB_NAME}"
 
# Unarchive library, then configure and make for specified architectures
# 编译静态链接库的函数
configure_make()
{
   ARCH=$1; GCC=$2; SDK_PATH=$3;
   LOG_FILE="${TEMP_LIB_PATH}-${ARCH}.log"
   tar xfz "${LIB_NAME}.tar.gz"
   pushd .; cd "${LIB_NAME}";

   ./Configure BSD-generic32 --openssldir="${TEMP_LIB_PATH}-${ARCH}" &> "${LOG_FILE}"
   
   make CC="${GCC} -arch ${ARCH}" CFLAG="-isysroot ${SDK_PATH}" &> "${LOG_FILE}"; 
   make install &> "${LOG_FILE}";
   popd; rm -rf "${LIB_NAME}";
}

# 分别开始编译四个平台架构的静态链接库
for ((i=0; i < ${#ARCHS[@]}; i++))
do
   # 获取 SDK 路径
   SDK_PATH=$(xcrun -sdk ${SDKS[i]} --show-sdk-path)
   # 过去 gcc 编译器路径
   GCC=$(xcrun -sdk ${SDKS[i]} -find gcc)
   # 编译
   configure_make "${ARCHS[i]}" "${GCC}" "${SDK_PATH}"
done

# Combine libraries for different architectures into one
# Use .a files from the temp directory by providing relative paths
# 通过 lipo 命令将四个平台架构的静态库打包成一个静态库
create_lib()
{
   LIB_SRC=$1; LIB_DST=$2;
   LIB_PATHS=( "${ARCHS[@]/#/${TEMP_LIB_PATH}-}" )
   LIB_PATHS=( "${LIB_PATHS[@]/%//${LIB_SRC}}" )
   lipo ${LIB_PATHS[@]} -create -output "${LIB_DST}"
}
mkdir "${LIB_DEST_DIR}";
create_lib "lib/libcrypto.a" "${LIB_DEST_DIR}/libcrypto.a"
create_lib "lib/libssl.a" "${LIB_DEST_DIR}/libssl.a"
 
# Copy header files + final cleanups
mkdir -p "${HEADER_DEST_DIR}"
cp -R "${TEMP_LIB_PATH}-${ARCHS[0]}/include" "${HEADER_DEST_DIR}"
rm -rf "${TEMP_LIB_PATH}-*" "{LIB_NAME}"
```

这个脚本的用法就是将脚本和刚刚下载的 OpenSSL 源码压缩包放在同一个目录下，然后不用解压 OpenSSL 压缩包，直接运行脚本即可。

```sh
sh  build-openssl.sh
```

耐心等待片刻之后，如果没有任何报错信息，则会在脚本所在目录多出两个目录 `include` 和 `lib`，在 `lib` 目录下就是我们刚刚通过脚本生成好的静态链接库 `libcrypto.a` 和 `libssl.a`。

##编译生成 Android 平台下适用的 OpenSSL 静态链接库

接下来我们继续编译 Android 平台下的 OpenSSL 静态链接库。编译 Android 下的静态链接库自然要用到 NDK。所以首先要确保你下载了 NDK。我这里使用的是 `android-ndk-r10d`。

同样根据平台架构不同 Android 下面可以生成 `arm`、`armv7`和`x86`。具体的生成步骤都是直接在命令行下执行的。

首先解压缩你的 OpenSSL 源代码压缩包并跳转至解压缩后的源代码目录.

```sh
tar -zxvf openssl-1.0.2c.tar.gz
cd openssl-1.0.2c
```

###armv7a

```sh
#设置你自己的 NDK 路径
export NDK=/Your Android NDK Path/android-ndk-r10d
$NDK/build/tools/make-standalone-toolchain.sh --platform=android-9 --toolchain=arm-linux-androideabi-4.6 --install-dir=`pwd`/android-toolchain-arm
export TOOLCHAIN_PATH=`pwd`/android-toolchain-arm/bin
export TOOL=arm-linux-androideabi
export NDK_TOOLCHAIN_BASENAME=${TOOLCHAIN_PATH}/${TOOL}
export CC=$NDK_TOOLCHAIN_BASENAME-gcc
export CXX=$NDK_TOOLCHAIN_BASENAME-g++
export LINK=${CXX}
export LD=$NDK_TOOLCHAIN_BASENAME-ld
export AR=$NDK_TOOLCHAIN_BASENAME-ar
export RANLIB=$NDK_TOOLCHAIN_BASENAME-ranlib
export STRIP=$NDK_TOOLCHAIN_BASENAME-strip
export ARCH_FLAGS="-march=armv7-a -mfloat-abi=softfp -mfpu=vfpv3-d16"
export ARCH_LINK="-march=armv7-a -Wl,--fix-cortex-a8"
export CPPFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 "
export CXXFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 -frtti -fexceptions "
export CFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 "
export LDFLAGS=" ${ARCH_LINK} "
./Configure android-armv7
PATH=$TOOLCHAIN_PATH:$PATH make
```

上述命令运行完成后，同样你会在 OpenSSL 源代码目录发现新生成的两个静态链接库文件`libcrypto.a` 和 `libssl.a`。我们新建一个目录 `armeabi-v7a` 将两个新生成的静态链接库文件移动到这个文件夹中备用。

然后我们删除掉刚刚解压缩的 OpenSSL 源码目录，重新解压缩 OpenSSL 的源代码压缩包，准备继续编译生成另外平台架构的静态链接库。

###arm

```sh
#设置你自己的 NDK 路径
export NDK=/Your Android NDK Path/android-ndk-r10d
$NDK/build/tools/make-standalone-toolchain.sh --platform=android-9 --toolchain=arm-linux-androideabi-4.6 --install-dir=`pwd`/android-toolchain-arm
export TOOLCHAIN_PATH=`pwd`/android-toolchain-arm/bin
export TOOL=arm-linux-androideabi
export NDK_TOOLCHAIN_BASENAME=${TOOLCHAIN_PATH}/${TOOL}
export CC=$NDK_TOOLCHAIN_BASENAME-gcc
export CXX=$NDK_TOOLCHAIN_BASENAME-g++
export LINK=${CXX}
export LD=$NDK_TOOLCHAIN_BASENAME-ld
export AR=$NDK_TOOLCHAIN_BASENAME-ar
export RANLIB=$NDK_TOOLCHAIN_BASENAME-ranlib
export STRIP=$NDK_TOOLCHAIN_BASENAME-strip
export ARCH_FLAGS="-mthumb"
export ARCH_LINK=
export CPPFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 "
export CXXFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 -frtti -fexceptions "
export CFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 "
export LDFLAGS=" ${ARCH_LINK} "
./Configure android
PATH=$TOOLCHAIN_PATH:$PATH make
```

然后我们新建一个目录 `armeabi` 将新生成的静态链接库文件 `libcrypto.a` 和 `libssl.a` 移动到这个文件夹中备用。删除掉源代码目录，重新解压，继续编译生成静态链接库。

###x86

```sh
#设置你自己的 NDK 路径
export NDK=/Your Android NDK Path/android-ndk-r10d
$NDK/build/tools/make-standalone-toolchain.sh --platform=android-9 --toolchain=x86-4.6 --install-dir=`pwd`/android-toolchain-x86
export TOOLCHAIN_PATH=`pwd`/android-toolchain-x86/bin
export TOOL=i686-linux-android
export NDK_TOOLCHAIN_BASENAME=${TOOLCHAIN_PATH}/${TOOL}
export CC=$NDK_TOOLCHAIN_BASENAME-gcc
export CXX=$NDK_TOOLCHAIN_BASENAME-g++
export LINK=${CXX}
export LD=$NDK_TOOLCHAIN_BASENAME-ld
export AR=$NDK_TOOLCHAIN_BASENAME-ar
export RANLIB=$NDK_TOOLCHAIN_BASENAME-ranlib
export STRIP=$NDK_TOOLCHAIN_BASENAME-strip
export ARCH_FLAGS="-march=i686 -msse3 -mstackrealign -mfpmath=sse"
export ARCH_LINK=
export CPPFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 "
export CXXFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 -frtti -fexceptions "
export CFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 "
export LDFLAGS=" ${ARCH_LINK} "
./Configure android-x86
PATH=$TOOLCHAIN_PATH:$PATH make
```
同样我们新建一个目录 `x86` 用来存放新生成的静态链接库文件 `libcrypto.a` 和 `libssl.a`。

##将 OpenSSL 接入 Cocos2d-x 项目

iOS 和 Android 适用的 OpenSSL 静态链接库我们都已经编译生成了。下面看看我们怎么将 OpenSSL 静态链接库接入到 Cocos2d-x 项目中来使用。

###引入 iOS 适用的 OpenSSL 静态链接库

首先这里要说明的是我使用的 Cocos2d-x 版本是 3.6。我们先在 Cocos2d-x  项目中新建一个文件夹。

```sh
mkdir -p Your_Cocos2d-x_Project_Path/Classes/security/openssl
```

然后我们将 iOS 适用的 OpenSSL 静态链接库文件 `libcrypto.a` 和 `libssl.a` 拷贝到我们刚刚新建的目录下。

在 Xcode 中将 OpenSSL 的静态链接库文件`libcrypto.a` 和 `libssl.a` 引入到项目中。具体做法就是在 `[Build Phases] ====> [Link Binary With Libraries]` 中添加对两个静态链接库的引用。

###在 Xcode 中添加 OpenSSL 头文件搜索路径

然后我们再新建一个用于存放 OpenSSL 头文件的文件夹

```sh
mkdir -p Your_Cocos2d-x_Project_Path/Classes/security/openssl/include/openssl
```

将我们刚刚生成 iOS 适用的 OpenSSL 静态链接库文件时生成的 `include` 文件夹下面找到的所有的 `.h` 头文件全部复制到我们刚刚生成的目录下面。

**这里需要特别注意**的是包含 OpenSSL 头文件的文件夹必须叫作 **openssl** ，否则项目编译会不成功。

然后继续在 Xcode 中设置 OpenSSL 头文件的搜索路径。具体做法就是在 `[Build Settings] ====> [User Header Search Paths]` 中添加刚刚我们建立的用于存放 OpenSSL 头文件的目录的路径。要添加的路径为 `Your_Cocos2d-x_Project_Path/Classes/security/openssl/include`

###引入 Android 适用的 OpenSSL 静态链接库

我们把刚刚用于存放 OpenSSL 静态链接库文件的 `armeabi` 文件夹拷贝到 Cocos2d-x 项目中来，拷贝的位置是 Android 用于存放库文件的 `libs` 文件夹。

```sh
cp -ivr Your_OpenSSL_Android_Path/armeabi  \
	Your_Cocos2d-x_Project_Path/proj.android/libs/
```

**这里需要特别注意**如果你的 Cocos2d-x 项目原本已经有了 `armeabi` 文件夹，注意不要覆盖，而是将 `libcrypto.a` 和 `libssl.a` 文件直接放入已有的 `armeabi` 文件夹中即可。

###在 Android.mk 文件中添加 OpenSSL 头文件搜索路径

用趁手的编辑器打开 `Your_Cocos2d-x_Project_Path/proj.android/jni/Android.mk` 文件，将 OpenSSL 头文件路径添加到 `LOCAL_C_INCLUDES` 变量中。

```
LOCAL_C_INCLUDES := \
	# ... Some Other Paths \
	$(LOCAL_PATH)/../../Classes/security/openssl/include \
	# ... Some Other Paths \
```

##走个捷径

如果你觉得编译太麻烦，要编译的东西太多太复杂，或者手头没有编译环境，翻墙下载个 NDK 又太费劲巴拉巴拉......总之你不想自己编译 OpenSSL 的静态链接库，那么你可以走一个捷径。直接使用我已经编译好的文件即可。[我已经把编译好的 OpenSSL 静态链接库放在 GitHub 上方便大家自取了](https://github.com/leenjewel/openssl_for_ios_and_android)。

```sh
git clone git@github.com:leenjewel/openssl_for_ios_and_android.git
```

##参考资料

如果你的 Cocos2d-x 项目编译运行成功，那么恭喜你，OpenSSL 库已经接入到你得项目中了，你已经可以调用 OpenSSL 的 API 来实现你自己的需求了。这里附上两个相关的资料供你参考：

>《How-To-Build-openssl-For-iOS》

>《Compiling the latest OpenSSL for Android》