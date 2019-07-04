---
layout:     post
title:      "Android NDK开发之旅--JNI数组的处理"
subtitle:   ""
date:       2019-06-25 11:36:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### 数组的处理（主要是同步问题）

Java声明如下：

```java
    public native void giveArray(int[] array);
```

C代码如下：

```c
//排序规则，小的在前
int compare(int *a, int *b) {
    return (*a) - (*b);
}
//传入
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_giveArray
(JNIEnv *env, jobject jobj, jintArray arr) {
    //qsort();
    //jintArray -> jint指针 ->c int 数组
    jint *elems = (*env)->GetIntArrayElements(env, arr, NULL);

    //数组的长度
    int len = (*env)->GetArrayLength(env, arr);

       //排序
    qsort(elems,len,sizeof(jint), compare);

    //C中操作同步到java,并释放资源
    (*env)->ReleaseIntArrayElements(env, arr, elems, 0);

}
```

最后在Java中测试：.

```c
   public native void giveArray(int[] array);

    public static void main(String[] args) {
 
        JniTest jniTest = new JniTest();
        int[] array = {100, 3, 10, 7, 5, 103, 160, 79, 51};
        jniTest.giveArray(array);
        for(int i : array){
            System.out.println(i);
        }
    }

结果输出：

3
5
7
10
51
79
100
103
160
```

#### 注意：

- ##### 通过GetIntArrayElements拿到C类型的数组的指针，然后才能进行C数组的处理。

- ##### C拿到Java的数组进行操作或者修改以后，需要调用ReleaseIntArrayElements进行更新，这时候Java的数组也会同步更新过来。

这个方法的最后一个参数是模式：

| 模式         | 作用                                    |
| ---------- | ------------------------------------- |
| 0          | Java数组进行更新，并且释放C/C++数组。               |
| JNI_ABORT  | Java数组不进行更新，但是释放C/C++数组。              |
| JNI_COMMIT | Java数组进行更新，不释放C/C++数组（函数执行完，数组还是会释放）。 |

C代码如下：

```c
JNIEXPORT jintArray JNICALL Java_com_haocai_jni_JniTest_getArray
(JNIEnv *env, jobject jobj, jint len) {

    //创建一个指定大小的数组
    jintArray jint_arr = (*env)->NewIntArray(env, len);
    jint *elems = (*env)->GetIntArrayElements(env, jint_arr, NULL);

    int i = 0;
    for (; i < len; i++) {
        elems[i] = i;
    }


    
    (*env)->ReleaseIntArrayElements(env, jint_arr, elems, 0);

    return jint_arr;
}
```

最后在Java中测试：.

```java
    public native void giveArray(int[] array);

    public static void main(String[] args) {
 
        JniTest jniTest = new JniTest();
        int[] array2 =  jniTest.getArray(5);
        for(int i : array2){
            System.out.println(i);
        }
    }

结果输出：
0
1
2
3
4
```

```c
(*env)->ReleaseIntArrayElements(env, jint_arr, elems, JNI_COMMIT);
结果输出：

0
1
2
3
4
```

```c
(*env)->ReleaseIntArrayElements(env, jint_arr, elems, JNI_ABORT); 或者注释该行
结果输出：

0
0
0
0
0
```












