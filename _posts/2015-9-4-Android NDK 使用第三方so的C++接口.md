---
layout:     post
title:      "Android NDK 使用第三方 so 库接口"
categories: android
date:       2015-9-4
---

在 Android 中使用 C/C++ 编程，一般是通过 Google 的 NDK（Native Development Kit）将 C/C++ 编译成动态链接库（.so文件）打包入 APK 中。
有时基于逻辑共享、跨平台、使用第三方库等考虑需要使用他方提供的 so 接口，这里主要介绍如何使用第三方 so 库的接口。

## 调用第三方so库 Java 接口

我们知道在 Java 层调用本地接口是通过 JNI（Java Native Interface）实现的。先在 Java 中声明一个 native 方法，然后用 **javah** 命令生成对应的 `.h` 头文件，
再用 C++ 实现 `.h` 中对应的接口。用 NDK 生成相应平台的 so 文件后，在 Java 中就可以直接调用之前声明的 native 方法了。

因此，要调用第三方 so 库中的 JNI 接口，首先需要有 声明了相应 native 方法的 Java文件。假设有个第三方 so 库：libhello-jni.so，实现的这样的 JNI 接口：

```cpp
jstring Java_com_example_hellojni_HelloJni_stringFromJNI( JNIEnv* env, jobject )
{
    return (*env)->NewStringUTF(env, "Hello Jni!");
}
```

那么，我们需要声明了此 native 方法的 Java 文件：HelloJni.java ，内容类似：

```java
package com.example.hellojni;

public class HelloJni {
    public native String stringFromJNI();
}
```

如果第三方没有提供这样的 java 文件，我们需要根据 C++ 中 JNI 接口的声明的样式，手动创建在对应的包下的声明相应的 native 的 java 文件。之后可直接在 java 中调用 `stringFromJNI` 接口。

## 调用第三方so库 C++ 接口

要调用第三方so库 C++ 接口，需要有声明了相应类或函数的 `.h` 头文件，然后还要设置 `Android.mk`、`Application.mk` 告诉 NDK 如何编译和使用第三方 so 库。
假设现在我们现在有第三方 so 库：libhello-jni.so，且此 so 有四种 CPU 架构的版本，如下图示：

<img src="/assets/img/2015-09-04-ndk-sofiles.png" alt="so文件" style="width: 40%;">

在 Android 工程下的 `jni` 目录新建文件夹 **prebuild** 用来处理第三方 so 库。将第三方 so 库的所有版本都放在 prebuild 目录下，并新建 **include** 文件夹，
将第三方提供的 so 库对应的头文件放入其中.同时在 prebuild 目录中创建 **Android.mk** 来处理第三方 so 库，此时 jni 的目录结构应类似：

<img src="/assets/img/2015-09-05-ndk-jnidir.png" alt="jni目录" style="width: 40%;">

我们先看下第三方库的头文件的声明：

```cpp
#ifndef __HELLOCPP_H__
#define __HELLOCPP_H__

#include <string>

class HelloCPP {
public:
    std::string hello();
};

#endif
```

在 `com_example_hellojni_ThirdPart.cpp` 中调用第三方 so 库的接口，如下：

```cpp
#include "com_example_hellojni_ThirdPart.h"
#include "prebuild/include/HelloCPP.h"
#include <string>
/*
 * Class:     com_example_hellojni_ThirdPart
 * Method:    stringFromJNI
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_example_hellojni_ThirdPart_stringFromJNI
  (JNIEnv * env, jobject)
{
	HelloCPP he;
    std::string cStr = he.hello();
    jstring jStr = env->NewStringUTF(cStr.c_str());
    return jStr;
}
```

这时运行程序肯定会找不到第三方库的接口，关键是要配置 `Android.mk`。jni根目录下的 `Android.mk` 配置如下：

```
MY_LOCAL_PATH := $(call my-dir)     # 定义MY_LOCAL_PATH变量为当前目录
LOCAL_PATH := $(MY_LOCAL_PATH)      # 赋值此构建本地路径

include $(CLEAR_VARS)               # 清除除了 LOCAL_PATH 外的变量

LOCAL_MODULE    := myhellojni       # 定义此模块名称
LOCAL_SRC_FILES := com_example_hellojni_ThirdPart.cpp  # 声明此模块使用的源文件
LOCAL_SHARED_LIBRARIES := prebuild  # 声明此模块使用的共享库

include $(BUILD_SHARED_LIBRARY)     # 声明此模块构建为共享库
include $(LOCAL_PATH)/prebuild/Android.mk              # 声明另一模块位置
```

其中，**LOCAL_MODULE** 定义了库的名称，如果生成的是共享库（即 BUILD_SHARED_LIBRARY）时会自动加上前缀 `lib` 和后缀 `.so`。 **LOCAL_SHARED_LIBRARIES** 中声明了构建第三方共享库的模块名，此模块名在`jni/prebuild/Android.mk` 中声明。在 `prebuild` 目录下的 Android.mk 配置如下：

```
LOCAL_PATH := $(MY_LOCAL_PATH)//prebuild               # 赋值此构建本地路径
include $(CLEAR_VARS)                                  # 清除除了 LOCAL_PATH 外的变量

LOCAL_MODULE := prebuild                               # 定义此模块名称
LOCAL_SRC_FILES := $(TARGET_ARCH_ABI)/libhello-jni.so  # 声明第三方库位置
LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH)/include       # 导出第三方库的头文件

include $(PREBUILT_SHARED_LIBRARY)                     # 声明此模块为预构建共享库
```

注意，这里赋值此构建本地路径时不能使用 `LOCAL_PATH := $(call my-dir)`，对于宏定义 `my-dir`，Google 官方文档有这么一个提醒：

>Due to the way GNU Make works, what this macro really returns is the path of the last makefile that the build system included when parsing the build scripts. For this reason, you should not call my-dir after including another file.

即，由于 GNU 的工作机制，宏变量 `my-dir` 代表解析构建脚本时最后包含此变量的 makefile 所在的路径，因此你不应在包含其他文件后调用 my-dir。
如果在 `include $(LOCAL_PATH)/prebuild/Android.mk` 指令后又调用了 my-dir 会导致此值变为 **jni/prebuild/** ，可能会导致找不到 so 文件等错误。
有多个 Android.mk 脚本时切记只在其中一个 Android.mk 中调用 my-dir，建议在 `jni` 目录下的 Androdid.mk 中调用并赋值给自定义变量 **MY_LOCAL_PATH**，
在其他文件中使用此变量做为 LOCAL_PATH。

在预构建脚本中 **LOCAL_SRC_FILES** 为第三方库的位置，为了构建时获取对应 cpu 架构的库文件应使用宏 **TARGET_ARCH_ABI** 来声明第三方库位置。这样在 ndk 构建时
能定位 LOCAL_PATH 下对应 cpu 构架名称的目录（如 armeabi/）下的库文件。`include $(PREBUILT_SHARED_LIBRARY)` 声明此模块为预构建共享库，
因此构建时不会生成 `libprebuild.so` 的库文件。

Application.mk 的配置比较简单，如下：

```
APP_ABI := armeabi armeabi-v7a mips x86  # 声明生成库文件的 cpu 架构
APP_STL := gnustl_static                 # 声明使用 GNU STL 静态库
NDK_TOOLCHAIN_VERSION := 4.8             # 声明构建使用的 GCC 版本
```

在 Android 工程的根目录下运行 `ndk-build`，或用 Eclipse 等 IED 进行构建，大功告成：

<img src="/assets/img/2015-09-05-ndk-build.png" alt="ndk构建" style="width: 100%;">

## 参考
1. C++ Library Support：http://developer.android.com/ndk/guides/cpp-support.html
1. Android.mk：http://developer.android.com/ndk/guides/android_mk.html
1. Using Prebuilt Libraries：http://developer.android.com/ndk/guides/prebuilts.html
