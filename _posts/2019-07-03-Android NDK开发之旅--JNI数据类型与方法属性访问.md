---
layout:     post
title:      "Android NDK开发之旅--JNI数据类型与方法属性访问"
subtitle:   ""
date:       2019-07-03 14:21:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### JNI数据类型

##### JNI的数据类型包含两种: 基本类型和引用类型

#### 基本类型

基本类型主要有jboolean, jchar, jint等, 它们和Java中的数据类型对应关系如下表所示:

| Java类型  | JNI类型    | 描述       |
| ------- | -------- | -------- |
| boolean | jboolean | 无符号8位整型  |
| byte    | byte     | 无符号8位整型  |
| char    | jchar    | 无符号16位整型 |
| short   | jshort   | 有符号16位整型 |
| int     | jint     | 32位整型    |
| long    | jlong    | 64位整型    |
| float   | jfloat   | 32位浮点型   |
| double  | jdouble  | 64位浮点型   |
| void    | void     | 无类型      |

#### 引用类型(对象)

JNI中的引用类型主要有类, 对象和数组. 它们和Java中的引用类型的对应关系如下表所示:

| Java类型    | JNI类型         | 描述        |
| --------- | ------------- | --------- |
| Object    | jobject       | Object类型  |
| Class     | jclass        | Class类型   |
| String    | jstring       | String类型  |
| Object[]  | jobjectArray  | 对象数组      |
| boolean[] | jbooleanArray | boolean数组 |
| byte[]    | jbyteArray    | byte数组    |
| char[]    | jcharArray    | char数组    |
| short[]   | jshortArray   | short数组   |
| int[]     | jintArray     | int数组     |
| long[]    | jlongArray    | long数组    |
| float[]   | jfloatArray   | float数组   |
| double[]  | jdoubleArray  | double数组  |
| Throwable | jthrowable    | Throwable |

#### native函数参数说明

每个native函数，都至少有两个参数（JNIEnv*,jclass或者jobject)。

1）当native方法为静态方法时：  
jclass 代表native方法所属类的class对象(JniTest.class)。

2）当native方法为非静态方法时：  
jobject 代表native方法所属的对象。

native函数的头文件可以自己写。



#### 关于属性与方法的签名

| 数据类型                 | 签名                      |
| -------------------- | ----------------------- |
| boolean              | Z                       |
| byte                 | B                       |
| char                 | C                       |
| short                | S                       |
| int                  | I                       |
| long                 | J                       |
| float                | F                       |
| double               | D                       |
| ully-qualified-class | Lfully-qualified-class; |
| type[]               | [type                   |
| method type          | (arg-types)ret-typ      |

##### 注意：

- ##### 类描述符开头的 'L' 与结尾的 ';' 必须要有

- ##### 数组描述符,开头的 '[' 必须要有

- ##### 方法描述符规则: "(各参数描述符)返回值描述符",其中参数描述符间没有任何分隔符号

从上表可以看出, 基本数据类型的签名基本都是单词的首字母大写, 但是boolean和long除外因为B已经被byte占用, 而long也被Java类签名的占用.

对象和数组的签名稍微复杂一些.

对象的签名就是对象所属的类签名, 比如String对象, 它的签名为Ljava/lang/String; .

数组的签名为[+类型签名, 例如byte数组. 其类型为byte, 而byte的签名为B, 所以byte数组的签名就是[B.同理可以得到如下的签名对应关系:

```cpp
char[]      [C
float[]     [F
double[]    [D
long[]      [J
String[]    [Ljava/lang/String;
Object[]    [Ljava/lang/Object;
```

方法签名具体方法：  
获取方法的签名比较麻烦一些，通过下面的方法也可以拿到属性的签名。  
打开命令行，输入javap，出现以下信息：

![](https://upload-images.jianshu.io/upload_images/1824809-d4da230405ec8d34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/544/format/webp)



上述信息告诉我们，通过以下命令就可以拿到指定类的所有属性、方法的签名了，很方便有木有？！

```cpp
javap -s -p 完整类名
```

我们通过cd命令，来到编译生成的class字节码文件目录（注意：非src目录。 eclipse 编译生成的class字节码文件在bin文件夹中， 而用idea编译器 编译生成的class字节码文件在out\production下），然后输入命令：

```
D:\IdeaProjects\jni1\out\production\jni1>javap -s -p com.haocai.jni.JniTest
```

得到以下信息：

```java
Compiled from "JniTest.java"
public class com.haocai.jni.JniTest {
  public java.lang.String key;
    descriptor: Ljava/lang/String;
  public static int count;
    descriptor: I
  public com.haocai.jni.JniTest();
    descriptor: ()V

  public static native java.lang.String getStringFromC();
    descriptor: ()Ljava/lang/String;

  public native java.lang.String getString2FromC(int);
    descriptor: (I)Ljava/lang/String;

  public native java.lang.String accessField();
    descriptor: ()Ljava/lang/String;

  public native void accessStaticField();
    descriptor: ()V

  public native void accessMethod();
    descriptor: ()V

  public native void accessStaticMethod();
    descriptor: ()V

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V

  public int genRandomInt(int);
    descriptor: (I)I

  public static java.lang.String getUUID();
    descriptor: ()Ljava/lang/String;

  static {};
    descriptor: ()V
}
```

##### 其中，descriptor：对应的值就是我们需要的签名了，注意签名中末尾的分号 ";" 不能省略。



### C/C++访问Java的属性、方法

在JNI调用中，肯定会涉及到本地方法操作Java类中数据和方法。在Java1.0中“原始的”Java到C的绑定中，程序员可以直接访问对象数据域。然而，直接方法要求虚拟机暴露他们的内部数据布局，基于这个原因，JNI要求程序员通过特殊的JNI函数来获取和设置数据以及调用java方法。



###### 有以下几种情况：

1.访问Java类的非静态属性。  
2.访问Java类的静态属性。  
3.访问Java类的非静态方法。  
4.访问Java类的静态方法。  
5.间接访问Java类的父类的方法。  
6.访问Java类的构造方法。



#### 一、访问Java的非静态属性

Java声明如下：

```java
 public String name = "kpioneer";

//访问非静态属性name，修改它的值 
//accessField 自定义的一个方法
public native void accessField();
```

C代码如下：

```c
//把java中的变量name中值kpioneer 变为kpioneer Goodman
JNIEXPORT jstring JNICALL Java_com_haocai_jni_JniTest_accessField
(JNIEnv *env, jobject jobject) {
    //Jclass 
    //jobj是t对象
    jclass cls = (*env)->GetObjectClass(env, jobject);
    //jfieldID
    //属性名称，属性签名
    jfieldID fid = (*env)->GetFieldID(env, cls, "name", "Ljava/lang/String;");


    //类似于反射
    //拿到jniTest(jobject) 中name的值
    /*
    Get<Type>Field：
    GetFloatField
    GetIntField
    GetLongField
    ...
    */
    jstring jstr = (*env)->GetObjectField(env, jobject, fid);
    printf("jstr:%#x\n", &jstr);
    //printf("jstr:%#x\n", &jstr);
    //jstring -> C 字符串

    boolean isCopy =NULL;
    //函数内部复制了，isCopy 为JNI_TRUE,没有复制JNI_FALSE
    char *c_str = (*env)->GetStringUTFChars(env, jstr, &isCopy );
    //意义：isCopy为JNI_FALSE,c_str和jstr都指向同一个字符串，不能修改java字符串

    char text[20] = " Goodman";
    strcat(c_str, text);//拼接函数

    //再把C字符串 ->jstring
    jstring new_str = (*env)->NewStringUTF(env, c_str);
    printf("jstr:%#x\n", &new_str);
    //修改name
    /*
    Set<Type>Field：
    SetFloatField
    SetIntField
    SetLongField
    ...
    */
    (*env)->SetObjectField(env, jobject, fid, new_str);

    //最后释放资源，通知垃圾回收器来回收
    //良好的习惯就是，每次GetStringUTFChars，结束的时候都有一个ReleaseStringUTFChars与之呼应
    (*env)->ReleaseStringUTFChars(env, jstr, c_str);
    return new_str;
}
```



最后在Java中测试：

```java
public static void main(String[] args) {
        JniTest jniTest = new JniTest();
        System.out.println("name修改前："+jniTest.name);
        jniTest.accessField();
        System.out.println("name修改后："+jniTest.name);
}

结果输出：
name修改前：kpioneer
name修改后：kpioneer Goodman
jstr:0x27cf238  //调用的C也打印输出
jstr:0x27cf2a8 
```



#### 二、访问Java的静态属性

Java声明如下

```java
public static int count = 9;
public native void accessStaticField();
```

```c
//访问静态属性
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_accessStaticField
(JNIEnv *env, jobject jobj) {
    //jclass
    jclass cls = (*env)->GetObjectClass(env, jobj);
    //jfieldID
    jfieldID fid =(*env)->GetStaticFieldID(env, cls, "count", "I");
    //GetStatic<Type>Field
    jint count = (*env)->GetStaticIntField(env, cls, fid);
    count++;
    //修改
    //SetStatic<Type>Field
    (*env)->SetStaticIntField(env, cls, fid, count);
}
```

最后在Java中测试：

```java
public static void main(String[] args) {
    JniTest jniTest= new JniTest();
    System.out.println("count修改前："+count);
    jniTest.accessStaticField();
    System.out.println("count修改后："+count);
}

结果输出：
count修改前：9
count修改后：10
```



#### 三、访问Java的非静态方法

Java声明如下：

```java
    //产生指定范围的随机数
    public int genRandomInt(int max){
        System.out.println("genRandomInt 执行了..");
        return new Random().nextInt(max);
    }
```

C代码如下：

```c
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_accessMethod
(JNIEnv *env, jobject jobj) {

    //Jclass
    jclass cls = (*env)->GetObjectClass(env, jobj);
    //JmethodID
    jfieldID mFid = (*env)->GetMethodID(env, cls, "genRandomInt", "(I)I");
    //调用
    //Call<Type>Method
    jint random = (*env)->CallIntMethod(env, jobj, mFid, 200);
    printf("random num:%ld",random);

}
```

最后在Java中测试：

```java
public static void main(String[] args) {
    JniTest jniTest= new JniTest();
    jniTest.accessMethod();
}
结果输出：
genRandomInt 执行了..

random num:109
```

#### 


```java
    public static  String getUUID(){
      return  UUID.randomUUID().toString();
    }
```

C代码如下：

```c
//访问Java静态方法
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_accessStaticMethod
(JNIEnv *env, jobject jobj) {

    //Jclass
    jclass cls = (*env)->GetObjectClass(env, jobj);
    //JmethodID
    jfieldID mFid = (*env)->GetStaticMethodID(env, cls, "getUUID", "()Ljava/lang/String;");

    //调用
    //CallStatic<Type>Method
    jstring uuid = (*env)->CallStaticObjectMethod(env, jobj, mFid);
    
    //随机文件名称 uuid.txt
    //jstring -> char*
    //isCopy JNI_FALSE,代表java和c操作的是同一个字符串
    char *uuid_str = (*env)->GetStringUTFChars(env, uuid, NULL);
    //拼接
    char filename[100];
    sprintf(filename, "D://%s.txt", uuid_str);
    FILE *fp = fopen(filename, "w");
    fputs("i love kpioneer", fp);
    fclose(fp);


}
```

最后在Java中测试：

```java
public static void main(String[] args) {
        JniTest jniTest = new JniTest();

        jniTest.accessStaticMethod();.
}
```

最终在D盘目录下生成名为2fbf3e41-741b-4899-8e4e-a6a80a23a0b2（UUID随机生成） 的txt文件

![](https://upload-images.jianshu.io/upload_images/1824809-9c8b8b4c43e79053.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/938/format/webp)



#### 五、访问Java类的构造方法

Java声明如下：

```java
   public native long accessConstructor();
```

C代码如下：

```c
//访问Java类的构造方法
//使用java.util.Date产生一个当前的事件戳
JNIEXPORT jlong  JNICALL Java_com_haocai_jni_JniTest_accesssConstructor
(JNIEnv *env, jobject jobj) {

    jclass cls = (*env)->FindClass(env, "java/util/Date");
    //jmethodID
    jmethodID  constructor_mid= (*env)->GetMethodID(env, cls,"<init>","()V");
    //实例化一个Date对象(可以在constructor_mid后加参)
    jobject date_obj =  (*env)->NewObject(env, cls, constructor_mid);
    //调用getTime方法
    jmethodID mid = (*env)->GetMethodID(env, cls, "getTime", "()J");
    jlong time = (*env)->CallLongMethod(env, date_obj, mid);

    printf("time:%lld\n",time);

    return time;

}
```

最后在Java中测试：

```java
public static void main(String[] args) {

        JniTest test = new JniTest();
        //直接在Java中构造Date然后调用getTime
        Date date = new Date();
        System.out.println(date.getTime());
        //通过C语音构造Date然后调用getTime
        long time = jniTest.accessConstructor();
        System.out.println(time);
}


结果输出：
1509688828013
1509688828013

time:1509688828013
```



#### 六、间接访问Java类的父类的方法

Java代码如下：  
父类：

```java
public class Human {
    public void sayHi(){
        System.out.println("人类打招乎(父类)");
    }
}
```

子类：

```java
public class Man extends Human {
    @Override
    public void sayHi() {
        System.out.println("男人打招乎");
    }
}
```

在TestJni类中有Human方法声明：

```java
    public Human human = new Man();

    public native void accessNonvirtualMethod();
```

如果是直接使用human .sayHi()的话，其实访问的是子类Man的方法  
但是通过底层C的方式可以间接访问到父类Human的方法，跳过子类的实现，甚至你可以直接哪个父类（如果父类有多个的话），这是Java做不到的。

下面是C代码实现，无非就是属性和方法的访问

```c
//调用父类的方法
JNIEXPORT void JNICALL Java_com_haocai_jni_JniTest_accessNonvirtualMethod
(JNIEnv *env, jobject jobj) {

    jclass cls = (*env)->GetObjectClass(env, jobj);

    //获取man属性(对象)
    jfieldID fid = (*env)->GetFieldID(env, cls, "human", "Lcom/haocai/jni/Human;");
    //获取
    jobject human_obj = (*env)->GetObjectField(env, jobj, fid);

    //执行sayHi方法
    jclass human_cls = (*env)->FindClass(env, "com/haocai/jni/Human");
    jmethodID mid = (*env)->GetMethodID(env, human_cls, "sayHi", "()V");
    
    //执行Java相关的子类方法
    (*env)->CallObjectMethod(env, human_obj, mid);

    //执行Java相关的父类方法
    (*env)->CallNonvirtualObjectMethod(env, human_obj, human_cls, mid);

}
```

1.当有这个类的对象的时候，使用(*env)->GetObjectClass()，相当于Java中的test.getClass()  
2.当有没有这个类的对象的时候，(*env)->FindClass()，相当于Java中的Class.forName("com.test.TestJni")  
这里直接使用CallVoidMethod，虽然传进去的是父类的Method ID，但是访问的让然是子类的实现。

最后，通过CallNonvirtualVoidMethod，访问不覆盖的父类方法（C++使用virtual关键字来覆盖父类的实现），当然你也可以指定哪个父类（如果有多个父类的话）。

最后在Java中测试：

```java
    public static void main(String[] args) {
        JniTest jniTest = new JniTest();
        jniTest.human.sayHi();
        jniTest.accessNonvirtualMethod();

    }

结果输出：
男人打招乎
男人打招乎  
人类打招乎(父类)
```

#### 实际案例---用JNI方法和属性访问解决中文编码乱码问题

中文乱码

```c
    char *cOutStr = "李四";
    string jstr = (*env)->NewStringUTF(env, cOutStr);
    return jstr; 直接返回会有中文乱码问题
```

###### 原因分析，调用NewStringUTF的时候，产生的是UTF-16的字符串，但是我们需要的时候UTF-8字符串。

##### 如果使用C语言方法解决中文编码问题，代码行数多（几百行+），且容易产生问题。所以直接通过Java 中的String(byte bytes[],String charsetName)构造方法来进行字符集变换，解决该问题。

Java声明如下：

```java
  public native String chineseChars(String str);
```

C代码如下：

```c
 JNIEXPORT jstring JNICALL Java_com_haocai_jni_JniTest_chineseChars
(JNIEnv *env, jobject jobj,jstring in) {

//输出
    char *cStr = (*env)->GetStringUTFChars(env, in, JNI_FALSE);
    printf("C %s\n", cStr);


    //c -> jstring
    char *cOutStr = "李四";
    //jstring jstr = (*env)->NewStringUTF(env, cOutStr);
    //return jstr; 直接返回会有中文乱码问题

    //解决中文乱码问题
    //执行java 中String(byte bytes[],String charsetName);
    //1.jmethodID
    //2.byte数组
    //3.字符编码

    jstring str_cls = (*env)->FindClass(env, "java/lang/String");
    //构造方法用<init>
    jmethodID construvtor_mid =  (*env)->GetMethodID(env, str_cls, "<init>", "([BLjava/lang/String;)V");

    //jbyte-> char
    //jbyteArray -> char[]
    jbyteArray bytes = (*env)->NewByteArray(env, strlen(cOutStr));
    //byte数组赋值
    //从0到strlen(cOutStr)，从头到尾
    (*env)->SetByteArrayRegion(env,bytes,0,strlen(cOutStr), cOutStr);

    //字符编码jstring
    jstring charsetName = (*env)->NewStringUTF(env, "GB2312");

    //调用构造函数，返回编码之后的jstring


    return (*env)->NewObject(env,str_cls, construvtor_mid,bytes, charsetName);

}
```

最后在Java中测试：

```java
   public static void main(String[] args) {
       JniTest jniTest = new JniTest();
       String outStr = jniTest.chineseChars("张三");
       System.out.println("中文输出："+outStr);
    }

结果输出：
中文输出：李四

C 张三
```

#### 总结

- ##### 1.C/C++完成的功能并不是所有代码一定要C/C++语句写，有时候C/C++可以调用现成的Java方法或属性解决问题，能起到事半功倍的作用。

- ##### 2.属性、方法的访问的使用是和Java的反射相类似。




