      

---
layout:     post
title:      "Android NDK开发十步流"
subtitle:   ""
date:       2019-07-02 16:06:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

     

附赠Android Studio 3.0 NDK配置的补充链接：[https://www.jianshu.com/p/433b2c93c6a7](https://www.jianshu.com/p/433b2c93c6a7)

     说到NDK，相信大家都不陌生，它是Google为便于Android开发提供的一种原生开发集：NativeNDKDevelopment Kit，而且也是一个包含API、构建工具、交叉编译、调试器、文档示例等一系列的工具集，可以帮助开发者快速开发C（或C++）的动态库，并能自动将so和Java应用一起打包成APK。  

与NDK密切相关的另一个词汇则是JNI，它是NDK开发中的枢纽，Java与底层交互绝大多数都是通过它来完成的，那么接下来看看什么是JNI?

JNI：Java Native Interface 也就是java本地接口，它是一个协议，这个协议用来沟通java代码和本地代码(c/c++)。通过这个协议，Java类的某些方法可以使用原生实现，同时让它们可以像普通的Java方法一样被调用和使用，而原生方法也可以使用Java对象，调用和使用Java方法。也就是说，使用JNI这种协议可以实现：java代码调用c/c++代码，而c/c++代码也可以调用java代码。



那为什么要使用NDK开发呢？

- 我们都知道，java是半解释型语言，很容易被反汇编后拿到源代码文件，在开发一些重要协议时，我们为了安全起见，使用C语言来编写这些重要的部分，来增大系统的安全性。

- 在一些复杂性的计算中，要求高性能的场景中，C/C++更加的有效率，代码也更便于复用。

当然还有其他的优点，这些都驱使我们选择相对来说高效和安全的NDK来开发我们的应用程序。

OK，说了那么多NDK，那到底怎么使用NDK来开发应用程序呢？

俗话说，工欲善其事必先利其器，想要使用NDK开发，必先打磨好工具。那下面首先来看看NDK的环境搭建吧。

### 


1.安装配置NDK

首先下载NDK，这里我使用的是Android-ndk-r14b-windows-x86_64，可以自主选择。

1). 解压NDK的zip包，注意路径目录不要出现空格和中文，这里建议大家把包解压到SDK目录里面，并命名为ndk-bundle，好处是，启动AS的时候会检查它并直接添加到ndk.dir中，减少我们的配置工作；

2). 配置path : 把解压好的路径添加到环境变量path中；

3).ndk-build：cd到解压后NDK的根目录，执行ndk-build命令。



给AS配置关联NDK，这里我使用的是androidstudio，使用Eclipse的会有所不同，请自行查找资料来配置。

1). 在建立的工程中的local.properties中添加如下配置   
ndk.dir=D:\guanmanman\androidStudio\sdk\ndk-bundle,这里注意下要使用转义字符“\”来进行字符转义。如果ndk目录是存放在SDK中，并命名为ndk-bundle，这个配置会自动为添加上去。



![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmotYOic0laiau4tvSg9qibhnaWDVbMGeqmGjvFnP4eYVXAVdGY2GW9P7opQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



2). 在工程中gradle.properties中添加对旧版本的NDK支持的配置   
android.useDeprecatedNdk=true

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmoKtRa9iaichUPELCO2RHT3ZQRf1kgwm2bhorP5edtVRrRqcTHgIM6guyQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 一、 layout布局

直接在layout中添加一个按钮Button控件，用于点击调用本地方法：

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmoB1ouia7o7JepjdIe4J2slMlQuT68hXJu4vZPbJubPgTbeYvD5ib9KMWw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 二 、在MainActivity中获取该控件并注册它的点击监听器

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmozViaYm7WYdpphfpWHlSV6ibibcpgejHGicuoMibqjB8QSyHW8XsGcNszPMw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

#### 三、 创建Java2CJNI类及本地方法

在我们的包下直接创建一个Java2CJNI类，并在类里创建一个java2C的本地方法

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmogXlhibs4KEIwb7nECD1UWA7EfyiamNXvLylsicic6vhVsXp9zyW22fctpg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 四 、通过javah命令获取到本地头文件

在项目根目录下，进入main->java目录，全选文件目录栏，直接输入cmd命令并按回车键进入docs命令，在命令中执行javah com.sanhui.ndkdemo.Java2CJNI命令：

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmoJdDL6z6SVUQmGecr26gKOjTLCnUj7Ogaib97wdevEhFmicPfeat2icoZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

执行完javah命令后，会在java当前目录下创建一个.h的头文件

#### 五、 在main目录下创建一个jni文件夹，并把（四）中的头文件转移到该文件夹下

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmoRL8dDtDsIyKbLeRkLww2x4zRiaN0PXtAEbEeLZF8qAjrJJBUTnZcadg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



打开该文件夹可以看到系统为我们创建好的本地方法头文件。



#### 六、 创建实现头文件的.C源文件

在jni目录下创建一个Java2C.c的源文件，通过#include引入我们的头文件com_sanhui_ndkdemo_Java2CJNI.h，并把在头文件下的声明方法JNIEXPORT jstring JNICALL Java_com_sanhui_ndkdemo_Java2CJNI_java2C(JNIEnv *, jobject);复制到我们的Java2C.c中，补全方法参数，并实现一个C字符串“I am From Native C .”的返回：

```cpp
JNIEXPORT jstring JNICALL Java_com_example_dyonline_chattool_Java2CJNI_java2c  
  (JNIEnv *env, jobject instance)  
 {  
        return env ->NewStringUTF("I am From Native C");  
 }

```



##### Android.mk (jni目录下)

LOCAL_MODULE : 表示模块的名称  
LOCAL_SRC_FILES: 参与编译的源文件

```c
# Copyright (C) 2009 The Android Open Source Project
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE    := helloJni
LOCAL_SRC_FILES := hello.cpp

include $(BUILD_SHARED_LIBRARY)
```

##### Application.mk(jni目录下)

> APP_ABI: 表示CPU的架构类型, 如armeabi, x86, mips

```
APP_ABI := armeabi
```









#### 七、在该Module下的build.gradle配置生成的so名称和支持的cpu类型

在android->defaultConfig下添加如下代码：

```java

ndk{
    moduleName "Java2C" //so文件名
    abiFilters  "armeabi-v7a"//CPU类型
    }
```

当然在这里不配置也是可以的，系统会用默认的项目名称作为so文件的名称，并且cpu也将会支持全部类型，只是当我们的项目名称改变的时候，在我们引用加载so文件的地方也需要改变，不改变的话就出现找不到so库的异常，所以，这里配置只是为了便利系统生成我们制定的so文件名，而不是根据项目名称生成。



在android下添加如下代码：

```
externalNativeBuild{  
    ndkBuild{  
        path "src/main/jni/Android.mk"  
  }  

}

```



#### 八 、加载so库

在我们创建的Java2CJNI类中加载so库，主要是为了在我们调用本地方法之前先编译本地源码。

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmokKVAKOwcmWbjf8sE3RIWPJyd4s7yCQxbwIAq0U859kStcJEMibCVk1Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



在使用 System.loadLibrary(“Java2C”);加载库时，库名一定要与在build.gradle中配置的moduleName 名称一致，否则将找不到库。





#### 九 、生成so文件

在项目的工具类中选择Build->Rebuild Project，进行重新编译工程，然后AS会为我们生成so文件，so文件所在目录为：NDKDemo\app\build\intermediates\ndk\debug\lib下

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmoNjxHdVq9WNm45gj0tn4nlQibjwCibrGTIwttuYib3icfd1DsKfuKUiavLaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)





注意：so文件命名方式是：lib+moduleName+.so

#### 十 、执行调用本地方法

在MainActivity中点击Button按钮调用本地方法。并通过Toast打印出来。

![](https://mmbiz.qpic.cn/mmbiz_png/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmorwGecfoWDCwcVuqqNZLZQz0CVcBSfOhoPvlImdbmicIwHzVLcD97W8A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



OK，到这里已经完成了一个DEMO级别的NDK应用开发了，那么来看看我们的执行结果：

![](https://mmbiz.qpic.cn/mmbiz_gif/2qPKlJe8EvqwumYSnTwNaCfAO3CyhPmo6GbqKGGX2Qw1jcbBm4oNvsH8ibvgD5Sp9WjCIPsDMk6p7ZnnjyavnuQ/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)





### 补充

#### 将 Gradle 关联到您的原生库

1. 从 IDE 左侧打开 **Project** 窗格并选择 **Android** 视图。
2. 右键点击您想要关联到原生库的模块（例如 **app** 模块），并从菜单中选择 **Link C++ Project with Gradle**
3. 从下拉菜单中，选择 **CMake** 或 **ndk-build**。
   1. 如果您选择 **CMake**，请使用 **Project Path** 旁的字段为您的外部 CMake 项目指定 `CMakeLists.txt` 脚本文件。
   2. 如果您选择 **ndk-build**，请使用 **Project Path** 旁的字段为您的外部 ndk-build 项目指定 [`Android.mk`](https://link.jianshu.com?t=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Fguides%2Fandroid_mk.html) 脚本文件。如果 [`Application.mk`](https://link.jianshu.com?t=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Fguides%2Fapplication_mk.html) 文件与您的 `Android.mk` 文件位于相同目录下，Android Studio 也会包含此文件。



![](https://upload-images.jianshu.io/upload_images/3371918-e03bf3e88d59614a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/980/format/webp)
