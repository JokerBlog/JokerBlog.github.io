---
layout:     post
title:      "Android NDK开发之旅--JNI引用"
subtitle:   ""
date:       2019-07-04 14:52:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### JNI引用

##### JNI引用概念：引用变量。

引用类型：局部引用和全局引用（全局引用里面包含全局弱引用）。

作用：在JNI中告知虚拟机何时回收一个JNI变量。

### 1.局部引用

局部引用，通过DeleteLocalRef手动释放对象。

典型使用场景：

访问一个很大的java对象，使用完之后，还要进行复杂的耗时操作。  
创建了大量的局部引用，占用了太多的内存，而且这些局部引用跟后面的操作没有关联性。  
例子：

Java代码声明：

```java
// 局部引用
public native void localRef();
```

C代码如下，这里模拟了局部引用的大量创建与删除：

```c
JNIEXPORT jintArray JNICALL Java_com_haocai_jni_JniTest_localRef
(JNIEnv *env, jobject jobj, jint len) {

    int i = 0;
    for (; i < 5;i++) {
        //创建Date对象
        jclass cls = (*env)->FindClass(env, "java/util/Date");
        jmethodID constructor_mid = (*env)->GetMethodID(env, cls, "<init>", "()V");
        jobject obj = (*env)->NewObject(env, cls, constructor_mid);
        //此处省略大量代码

        //不再使用jobject对象了
        //通知垃圾回收器回收这些对象
        (*env)->DeleteLocalRef(env, obj);
    }
}
```

java测试：

```java
public static void main(String[] args) {

    JniTest jniTest = new JniTest();
    jniTest.localRef();

}
```



#### 2.全局引用

主要作用：共享(可以跨多个线程)，手动控制内存使用。

Java中提供三个方法，分别用于创建、获取、删除JNI全局引用。

```java
public native void createGlobalRef();

public native String getGlobalRef();

public native void deteleGlobalRef();
```

C代码如下：

```c
//创建
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_createGlobalRef
(JNIEnv *env, jobject jobj, jint len) {
    jstring obj = (*env)->NewStringUTF(env, "jni development is powerful!");
    global_str = (*env)->NewGlobalRef(env, obj);
}
//获取
JNIEXPORT jstring JNICALL Java_com_haocai_jni_JniTest_getGlobalRef
(JNIEnv *env, jobject jobj, jint len) {

    return global_str;
}
//释放
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_deleteGlobalRef
(JNIEnv *env, jobject jobj, jint len) {

    (*env)->DeleteGlobalRef(env, global_str);

}
```

java测试：

```java
public static void main(String[] args) {

        JniTest jniTest = new JniTest();
        jniTest.createGlobalRef();
        System.out.println(jniTest.getGlobalRef());
        //用完之后释放
        jniTest.deleteGlobalRef();
        System.out.println("释放完了...");
        //删除之后再取出会抛出空指针异常
        System.out.println(jniTest.getGlobalRef());
}
```

//弱全局引用  
//节省内存，在内存不足时可以释放所引用的对象  
//可以引用一个不常用的对象，如果为NULL，临时创建  
//创建：NewWeakGlobalRef,销毁:DeleteGlobalWeakRef

//异常处理  
//1.保证Java代码可以运行  
//2.补救措施保证C代码继续运行

```c
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_exeception
(JNIEnv *env, jobject jobj, jint len) {
    jclass cls = (*env)->GetObjectClass(env, jobj);
    jfieldID fid = (*env)->GetFieldID(env, cls, "key2", "Ljava/lang/String;");
    //检测是否发生Java异常
    jthrowable execption = (*env)->ExceptionOccurred(env);
    if (execption!=NULL) {
        //让Java代码可以继续运行
        //清空异常信息
        (*env)->ExceptionClear(env);

        //补救措施
        fid = (*env)->GetFieldID(env, cls, "key", "Ljava/lang/String;");

    }
    //获取属性的值
    jstring jstr = (*env)->GetObjectField(env, jobj, fid);
    char *str = (*env)->GetStringUTFChars(env, jstr, NULL);

    //对比属性值是否合法  stricmp比较字符串s1和s2，但不区分字母的大小写
    if (_stricmp(str, "super kpioneer") != 0) {
        //认为抛出异常，给Java层处理
        jclass newExcCls = (*env)->FindClass(env, "java/lang/IllegalArgumentException");
        (*env)->ThrowNew(env, newExcCls, "key's value is invalid!");
    }


}
```

### 3.弱全局引用

使用方法与全局引用类似:

- 通过NewWeakGlobalRef创建全局引用。
- 通过DeleteWeakGlobalRef删除全局引用。

与全局引用不一样的是，弱全局引用特点：

- 节省内存，在内存不足时可以是释放所引用的对象。
- 可以引用一个不常用的对象，如果为NULL的时候（被回收了），可以手动再临时创建一个。

与全局引用类似，弱引用可以跨方法、线程使用。但与全局引用很重要不同的一点是，弱引用不会阻止 GC 回收它引用的对象。当本地代码中缓存的引用不一定要阻止 GC 回收它所指向的对象时，弱引用就是一个最好的选择。看下面的代码段:

```c
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_weakRef
(JNIEnv *env, jobject obj) {
    static jclass clz = NULL;
    if (clz == NULL) {
        jclass clzLocal = (*env)->GetObjectClass(env, obj);
        if (clzLocal == NULL) {
            return; /* 没有找到这个类 */
        }
        clz = (jclass)(*env)->NewWeakGlobalRef(env,clzLocal);
        if (clz == NULL) {
            return; /* 内存溢出 */
        }
    }
    /* 使用clz的引用 */
    if ((*env)->IsSameObject(env,clz, NULL)) {
        printf("clz is recycled\n");
    }
    else {
        printf("clz is not recycled\n");
    }

}
```

java测试：

```java
public native void weakRef();
public static void main(String[] args) {

        JniTest jniTest = new JniTest();
        jniTest.weakRef();
}

结果输出：
clz is not recycled
```

我们在使用弱引用的时候，必须先判断这个弱引用是指向一个活动的对象，还是一个已经被GC回收来的对象。NULL引用指向jvm中的null对象，可以通过IsSameObject来判断这引用是否被回收。通过env->IsSameObject(clz, NULL)来判断，被回收了则返回 JNI_TRUE（或者 1），否则返回 JNI_FALSE（或者 0）。

当我们的本地代码不再需要一个弱全局引用时，也应该调用 DeleteWeakGlobalRef 来释放它，如果不手动调用这个函数来释放所指向的对象，JVM 仍会回收弱引用所指向的对象，但弱引用本身在引用表中所占的内存永远也不会被回收。












