---
layout:     post
title:      "Android NDK开发之旅--JNI异常处理"
subtitle:   ""
date:       2019-07-04 15:03:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### 异常处理

异常测试例子：

```java
public native void testException1();

public static void main(String[] args) {

    JniTest test = new JniTest();

    try {
        test.testException();
        System.out.println("程序无法继续执行1，这句话不会被打印\n");
    } catch (Throwable t) {
        System.out.println("捕获到JNI抛出的异常(Throwable)，这句话会被打印" + t.getMessage() + "\n");
    }

    System.out.println("程序继续执行2，这句话会被打印\n");

}
```

C代码如下：

```c
//异常处理
JNIEXPORT void JNICALL Java_com_test_JniTest_testException1
(JNIEnv * env, jobject jobj){

    jclass clz=  (*env)->GetObjectClass(env, jobj);
    //属性名字不小心写错了，拿到的是空的jfieldID
    jfieldID fid = (*env)->GetFieldID(env, clz, "key1", "Ljava/lang/String;");

    //此处抛出的异常，Java可以通过Throwable来捕获

    printf("C can run , this will print");
    //这里竟然还可以继续执行
    jstring key =  (*env)->GetObjectField(env, jobj, fid);
    //遇到这句话的时候，C程序Crash了
    char* c_str = (*env)->GetStringUTFChars(env, key, NULL);
    printf("C could not run , this will not print");
}
```

通过例子可以知道，JNI层自己抛出的异常是Error类型，Java可以通过Throwable或者Error来捕获得到，捕获异常后Java代码可以继续执行下去。

##### 为了确保Java、C/C++代码可以正常执行下去，需要：

在JNI层手动清空异常信息（ExceptionClear），保证代码可以运行。  
补救措施保证C/C++代码继续运行。  
例如：

```c
//异常处理
JNIEXPORT void JNICALL Java_com_test_JniTest_testException1
(JNIEnv * env, jobject jobj){
    jclass clz = (*env)->GetObjectClass(env, jobj);
    //属性名字不小心写错了，拿到的是空的jfieldID
    jfieldID fid = (*env)->GetFieldID(env, clz, "key1", "Ljava/lang/String;");

    jthrowable err = (*env)->ExceptionOccurred(env);
    if (err != NULL){
        //手动清空异常信息，保证Java代码能够继续执行
        (*env)->ExceptionClear(env);
        //提供补救措施，例如获取另外一个属性
        fid = (*env)->GetFieldID(env, clz, "key", "Ljava/lang/String;");
    }


    jstring key = (*env)->GetObjectField(env, jobj, fid);
    char* c_str = (*env)->GetStringUTFChars(env, key, NULL);
}
```

测试代码如下：

```java
public static void main(String[] args) {

    JniTest test = new JniTest();

    try {
        test.testException();
        System.out.println("程序没有异常，这句话会被打印\n");
    } catch (Exception e) {
        System.out.println("没有捕获到JNI抛出的异常，这句话不会被打印" + e.getMessage() + "\n");
    }

    System.out.println("程序继续执行，这句话会被打印\n");

}
```

用户可以手动通过ThrowNew函数抛出异常，同样可以被Java代码捕获：

```c
//异常处理
JNIEXPORT void JNICALL Java_com_test_JniTest_testException
(JNIEnv * env, jobject jobj){
    jclass clz = (*env)->GetObjectClass(env, jobj);
    //属性名字不小心写错了，拿到的是空的jfieldID
    jfieldID fid = (*env)->GetFieldID(env, clz, "key1", "Ljava/lang/String;");

    jthrowable err = (*env)->ExceptionOccurred(env);
    if (err != NULL){
        //手动清空异常信息，保证Java代码能够继续执行
        (*env)->ExceptionClear(env);
        //提供补救措施，例如获取另外一个属性
        fid = (*env)->GetFieldID(env, clz, "key", "Ljava/lang/String;");
    }


    jstring key = (*env)->GetObjectField(env, jobj, fid);
    char* c_str = (*env)->GetStringUTFChars(env, key, NULL);

    //参数不正确，程序员自己抛出异常，可以在Java中捕获
    if (_stricmp(c_str,"efg") != 0){
        jclass err_clz = (*env)->FindClass(env, "java/lang/IllegalArgumentException");
        (*env)->ThrowNew(env, err_clz, "key value is invalid!");
    }
}
```

测试代码如下：

```java
public static void main(String[] args) {

    JniTest test = new JniTest();

    try {
        test.testException();
        System.out.println("JNI手动抛出了异常，Java不会继续执行，这句话不会被打印\n");
    } catch (Exception e) {
        System.out.println("捕获到JNI手动抛出的异常，这句话会被打印：" + e.getMessage() + "\n");
    }

    System.out.println("程序继续执行，这句话会被打印\n");

}
```

### 异常处理总结

> JNI自己抛出的异常，是Error类型，Java可以通过Throwable或者Error来捕获得到，捕获异常后Java代码可以继续执行下去。在C层可以清空（ExceptionClear），保证try中的代码Java代码继续执行，并且最好要提供补救措施，确保JNI层代码正常继续运行。  
> 用户通过ThrowNew手动抛出的异常，同样可以在Java层捕捉得到。












