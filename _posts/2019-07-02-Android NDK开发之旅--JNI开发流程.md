

---
layout:     post
title:      "Android NDK开发之旅--JNI开发流程"
subtitle:   ""
date:       2019-06-25 10:07:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

#### 引言

在学习了C语言基础之后 ，我们简单的了解了C语言编程的一些范式 ， 了解了指针 ， 结构体 ， 联合体 ， 函数 ， 文件IO等等 。我们最终的目的是要学会NDK开发 ， 而NDK开发就离不开我们的JNI技术 。下面 ， 就来开始我们的JNI之旅吧 。



#### JNI的概念

JNI全称 Java Native Interface ， java本地化接口 ， 可以通过JNI调用系统提供的API ， 我们知道 ， 不管是linux还是windows亦或是mac os ， 这些操作系统 ， 都是依靠C/C++写出来的 ， 还包括一些汇编语言写的底层硬件驱动 。java和C/C++不同 ， 它不会直接编译成平台机器码，而是编译成虚拟机可以运行的java字节码的.class文件，通过JIT技术即时编译成本地机器码，所以有效率就比不上C/C++代码，JNI技术就解决了这一痛点，下面我们来看看JNI调用示意图：

![](https://upload-images.jianshu.io/upload_images/1824809-3bae0b3309b73c3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/454/format/webp)



从上图可以得知 ，JNI技术通过JVM调用到各个平台的API ， 虽然JNI可以调用C/C++ ， 但是JNI调用还是比C/C++编写的原生应用还是要慢一点 ， 不过对高性能计算来说 ， 这点算不得什么 ， 享受它的便利 ， 也要承担它的弊端 。

#### JNI开发流程如下：

> 1.编写native方法  
> 2.javah方法，生成.h头文件  
> 3.复制.h头文件到CPP工程中  
> 4.复制jni.h和jni_md.h文件到CPP工程中  
> 5.实现.h头文件声明的函数  
> 6.生成dll文件  
> 7.java项目引入该dll文件  
> 8.加载动态链接库  
> 9.运行java类



#### 1.编写native方法

```java
public class JniTest {

    public native static String getStringFromC();

    public static void main(String[] args){
     String text= getStringFromC();
     System.out.printf(text);
    }

}
```

#### 2.调用java 中javah方法，生成.h头文件

打开命令行，通过cd命令转到当前Java工程的src目录下面，然后执行javah命令，参数是完整类名：

```java
D:\IdeaProjects\jni1\src>javah com.haocai.jni.JniTest
```

### [javah 命令找不到类文件的解决办法](https://www.cnblogs.com/maydow/p/4671320.html)

用javac  生成class文件后。

假设class文件在源文件目录的包下，先将编译路径设置成源文件目录即可

set classpath=G:\eclipse java\HelloWorld\src

然后

javah -jni com.example.helloworld.JNITest

搞定



##### 当文件中含有中文时可能出现

```
错误: 编码GBK的不可映射字符
```

所以可以指定编码格式，解决该问题

```
D:\IdeaProjects\jni1\src>javah -jni -encoding UTF-8  com.haocai.jni.JniTest
```

然后在src目录就会生成.h头文件：

![](https://upload-images.jianshu.io/upload_images/1824809-8dcd6be374209994.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600/format/webp)

#### 3.复制.h头文件到CPP工程中

将.h头文件复制到VS的代码文件目录下 ， 在解决方案中的头文件目录-> 右键-> 添加 -> 添加现有项 。 将我们的头文件添加进来。



#### 4.复制jni.h和jni_md.h文件到CPP工程中

```
# JDK 的jni.h头文件目录
D:\Java\jdk1.8.0_60\include\jni.h
# 在jni.h头文件中，又引入了jni_md.h头文件 ， 所有这个我们也要一并赋值过来
D:\Java\jdk1.8.0_60\include\win32\jni_md.h
```

将jni.h这个头文件按照上述步骤 ， 添加到头文件目录中 ， 注意将<>改成" " ， <>表示引入的是系统头文件，" "表示引入的是第三方头文件。

![](https://upload-images.jianshu.io/upload_images/1824809-be8e8c462c2427b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/382/format/webp)





#### 5.实现.h头文件中声明的函数。

com_haocai_jni_JniTest.h内容

```c
#include <jni.h>
/* Header for class com_haocai_jni_JniTest */

#ifndef _Included_com_haocai_jni_JniTest
#define _Included_com_haocai_jni_JniTest
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_haocai_jni_JniTest
 * Method:    getStringFromC
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_haocai_jni_JniTest_getStringFromC
  (JNIEnv *, jclass);

#ifdef __cplusplus
}
#endif
#endif
```

com_haocai_jni_JniTest.h中头文件函数

```c
/*
 * Class:     com_haocai_jni_JniTest
 * Method:    getStringFromC
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_haocai_jni_JniTest_getStringFromC
  (JNIEnv *, jclass);
```

##### 新建一个.c的文件 ，引入我们生成的头文件 ，然后实现我们生成的C语言函数。

```c
#include "com_haocai_jni_JniTest.h"

//函数实现
JNIEXPORT jstring JNICALL Java_com_haocai_jni_JniTest_getStringFromC
(JNIEnv *env, jclass jcls) {

    //JNIEnv 结构体指针
    //env二级指针
    //代表Java运行环境，调用Java中的代码

    //简单实现
    //将C的字符串转为一个java字符串
    return (*env)->NewStringUTF(env,"String From C");

}
```

#### 注意：JNI开发中JNIEnv在C和C++中实现的区别

JNIEnv：JNIEnv里面有很多方法，与Java进行交互，代表Java的运行环境。JNI Environment。

##### 在C中：

> JNIEnv 结构体指针的别名  
> env 二级指针

```c
JNIEXPORT jstring JNICALL Java_com_test_JniTest_getStringFromC
(JNIEnv * env, jclass jcls){
    //env是一个二级指针，函数中需要再次传入
    return (*env)->NewStringUTF(env, "String From C");
}
```

##### 在C++中：

JNIEnv 是一个结构体的别名  
env 一级指针

```c
JNIEXPORT jstring JNICALL Java_com_test_JniTest_getStringFromC
(JNIEnv * env, jclass jcls){
    //env是一个一级指针，函数中不需要再次传入
    return env->NewStringUTF("String From C");
}
```

为什么要用二级指针：

为什么需要传入JNIEnv？函数执行过程中需要JNIEnv

> C++为什么没有传入？因为C++中有this指针。  
> C++只是对C的那一套进行的封装，给一个变量赋值为指针，这个变量是二级指针  
> 源码分析

在jni.h头文件中有下面的预编译代码：

```c
#ifdef __cplusplus
typedef JNIEnv_ JNIEnv;
#else
typedef const struct JNINativeInterface_ *JNIEnv;
#endif
```

如果是C环境的话，JNIEnv就是一个JNINativeInterface_结构体的指针别名。  
如果是C++环境的话，JNIEnv就是一个结构体JNIEnv_的别名，而JNIEnv_是对JNINativeInterface_的封装。



##### C语言中结构体与指针相关知识回顾

```c
struct Man
{
    char name[20];
    int  age;
};

void main() {
    struct Man m1 = { "Jack",30 };
    //结构体指针
    struct Man *p = &m1;
    printf(" %s, %d\n", m1.name, m1.age);
    printf(" %s, %d\n",(*p).name,(*p).age);
    //"->"箭头是"(*P)."的简写
    printf(" %s, %d\n", p->name, p->age);

    system("pause");
}

结果输出：
 Jack, 30
 Jack, 30
 Jack, 30
```

##### C语言中NewStringUTF函数，大致实现如下：

```c
#include <stdio.h>

//JNIEnv结构体的指针别名
typedef struct JNINativeInterface_* JNIEnv;

//结构体
struct JNINativeInterface_
{
    //函数指针
    char* (*NewStringUTF)(JNIEnv*, char*);

};

//函数实现
char* NewStringUTF(JNIEnv* env,char* str) {
//NewStringUTF执行过程，仍然需要用到JNIEnvironment
    return str;
}


void main() {
    //实例化结构体
    struct JNINativeInterface_ struct_env;
    struct_env.NewStringUTF = NewStringUTF;

    //结构体指针
    JNIEnv e = &struct_env;

    //结构体的二级指针
    JNIEnv *env = &e;

    //通过二级指针调用函数
    char* str = (*env)->NewStringUTF(env, "abc");

    printf("%s", str);
    getchar();
}


    结果输出：
    abc
```

也就是说，我们在C语音调用下面这句的时候：

```c
(*env)->NewStringUTF(env, "String From C");
```

env是结构体的二级指针，它取内容*env是一级指针，通过一级指针就可以通过->符号操作结构体了。  
而NewStringUTF函数中需要用到JNIEnvironment，因此需要继续传入这个二级指针env自身。

但是在C++里面，JNIEnv就是一个结构体的别名，通过使用一级指针同样可以访问结构体本身，但是由于C++里面有this关键字代表自身，因此可以省略传入参数（已经封装好）。

```c
struct JNIEnv_ {
    const struct JNINativeInterface_ *functions;

    jstring NewStringUTF(const char *utf) {
        return functions->NewStringUTF(this,utf);
    }

    //代码省略
}
```

如上所示，C++中的JNIEnv就是JNIEnv_的别名，而JNIEnv_是对JNINativeInterface_的一次封装，在函数调用的时候，最终还是调用JNINativeInterface_结构体的方法：

```c
functions->NewStringUTF(this, utf);
```

其中，本来JNINativeInterface_的函数NewStringUTF就是需要传入二级指针的，因为C++中有this指针，代表着调用这functions的指针（其实就是二级指针），因此在C++中可以直接使用this指针代表当前调用者的指针（二级指针），而在C语音中就需要有二级指针去做了。



#### 6.生成动态链接库dll文件

动态库 

1.动态库把对一些库函数的链接载入推迟到程序运行的时期  
2.可以实现进程之间的资源共享。（因此动态库也称为共享库）  
3.可以动态注入到程序中 win(.dll)linux(.so) 



静态库 

1.静态库对函数库的链接是放在编译时期完成的。  
2.程序在运行时与函数库再无瓜葛，移植方便。  
3.浪费空间和资源，因为所有相关的目标文件与牵涉到的函数库被链接合成一个可执行文件。 win(.lib)linux(.a)



了解了静态库和动态库 ， 下面我们就来生成一个动态库，以VS为例：

```
选中项目 -> 右键 -> 属性 -> 常规 -> 项目默认值 -> 配置类型 , 选择动态库.dll
```



![](https://upload-images.jianshu.io/upload_images/1824809-9893f1a655b78a85.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/876/format/webp)

配置完成之后 ， 选中项目 -> 生成 。即可生成动态链接库.dll文件。

![](https://upload-images.jianshu.io/upload_images/1824809-dcfd9d64db88f535.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/628/format/webp)

#### 7.java项目引入该dll文件

![](https://upload-images.jianshu.io/upload_images/1824809-1d1ddca52abd3ab9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



#### 8.加载动态链接库

```java
 //加载动态库
    static{
        System.loadLibrary("jni1");
    }
```

#### 9.运行java类

```java
public class JniTest {

    public native static String getStringFromC();

    public static void main(String[] args){
     String text= getStringFromC();
     System.out.printf(text);
    }

    //加载动态库
    static{
        System.loadLibrary("jni1");
    }
}
```

#### 结语

##### 这篇是JNI系列的开篇 ， 其实流程比较简单。但是我们要了解JNIEnv其后的结构体与指针知识，为后续深入学习JNI打下基础。


