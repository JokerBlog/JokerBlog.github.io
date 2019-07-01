---
layout:     post
title:      "Android NDK开发之旅--NDK介绍"
subtitle:   ""
date:       2019-07-01 14:28:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### 一、NDK产生的背景

Android平台从诞生起，就已经支持C、C++开发。众所周知，Android的SDK基于Java实现，这意味着基于Android SDK进行开发的第三方应用都必须使用Java语言。但这并不等同于“第三方应用只能使用Java”。在Android SDK首次发布时，Google就宣称其虚拟机Dalvik支持JNI编程方式，也就是第三方应用完全可以通过JNI调用自己的C动态库，即在Android平台上，“Java+C”的编程方式是一直都可以实现的。

不过，Google也表示，使用原生SDK编程相比Dalvik虚拟机也有一些劣势，Android SDK文档里，找不到任何JNI方面的帮助。即使第三方应用开发者使用JNI完成了自己的C动态链接库（so）开发，但是so如何和应用程序一起打包成apk并发布？这里面也存在技术障碍。比如程序更加复杂，兼容性难以保障，无法访问Framework API，Debug难度更大等。开发者需要自行斟酌使用。

于是NDK就应运而生了。NDK全称是Native Development Kit。

NDK的发布，使“Java+C”的开发方式终于转正，成为官方支持的开发方式。NDK将是Android平台支持C开发的开端。



### 二、为什么使用NDK

1.代码的保护。由于apk的java层代码很容易被反编译，而C/C++库反汇难度较大。

2.方便借助第三方C/C++优秀开源库。

3.提高程序的执行效率。将要求高性能的应用逻辑使用C开发，从而提高应用程序的执行效率。

4.便于移植。用C/C++写得库可以方便在其他的嵌入式平台上再次使用。



### 三、NDK简介

1.NDK 即 Native Development Kit 是一系列工具的集合

NDK提供了一系列的工具，帮助开发者快速开发C（或C++）的动态库，并能自动将so和java应用一起打包成apk。这些工具对开发者的帮助是巨大的。

NDK集成了交叉编译器，并提供了相应的mk文件隔离CPU、平台、ABI等差异，开发人员只需要简单修改mk文件（指出“哪些文件需要编译”、“编译特性要求”等），就可以创建出so。

NDK可以自动地将so和Java应用一起打包，极大地减轻了开发人员的打包工作。

2.NDK提供了一份稳定、功能有限的API头文件声明

Google明确声明该API是稳定的，在后续所有版本中都稳定支持当前发布的API。从该版本的NDK中看出，这些API支持的功能非常有限，包含有：C标准库（libc）、标准数学库（libm）、压缩库（libz）、Log库（liblog）。

3.NDK 与 JNI  
JNI -- Java Native Interface （JNI就是Java调用C/C++的规范）  
NDK就是基于JNI



### 四、Android Studio环境NDK开发步骤

第一步：新建一个Android工程  
第二步：在AndroidStudio中配置NDK路径  
第三步：编译生成.class文件  
第四步：定义本地方法  
第五步：生成jni目录以及对应的.h头文件  
两种方式（通过工具生成、通过命令生成）  
cd app/src/main/java  
javah -d ../jni com.example.lixinyu.myapplication2.MainActivity  
第六步：配置build.gradle文件  
注：类似于Eclipse中的.mk文件  
第七步：配置local.properties文件  
注：指定NDK目录（一般情况下工具自动配置）  
第八步：配置gradle.properties文件  
android.useDeprecatedNdk=true  
注：支持低版本，否则编译不通过  
第九步：定义实现文件（.c或者.cpp文件）  
第十步：测试






