

---
layout:     post
title:      "Android NDK开发之旅--C语言(上)"
subtitle:   ""
date:       2019-07-01 16:27:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- NDK    
---

### C 语言包含的数据类型

如下图所示：

![](https://upload-images.jianshu.io/upload_images/1824809-e72434db3757adf7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/875/format/webp)



### C语言的基本数据类型：

##### short、int、long、char、float、double 这六个关键字代表C 语言里的六种基本数据类型。

格式化输出的时候：  
int %d  
short %d  
long %ld  
float %f  
double %lf  
char %c



%x 十六进制  
%o 八进制  
%s 字符串  
%p一般以十六进制整数方式输出指针的值，附加前缀0x

在32 位的系统上short 咔出来的内存大小是2 个byte；  
int 咔出来的内存大小是4 个byte；  
long 咔出来的内存大小是4 个byte；  
float 咔出来的内存大小是4 个byte；  
double 咔出来的内存大小是8 个byte；  
char 咔出来的内存大小是1 个byte。  
（注意这里指一般情况，可能不同的平台还会有所不同，具体平台可以用sizeof 关键字测试一下）



### 示例代码：

```c
//引入头文件
#include <stdlib.h>
#include <stdio.h>

void main(){

    int i;
    printf("请输入一个整数");
    scanf("%d", &i);

    printf("%d\n",i);
    float f = 10.01;
    printf("%f\n",f);

    //求某个类型所占的字节数，具体跟操作系统有关
    printf("int类型所占的字节数%d\n",sizeof(int));
    printf("float类型所占的字节数%d\n",sizeof(float));
    printf("double类型所占的字节数%d\n",sizeof(double));

    //循环的标准写法，循环变量需要抽取出来，否则在Linux环境下GCC下编译 报错
    int n = 0;
    for (;n<10;n++)
    {
        printf("%d\n",n);
    }

    //等待输入，目的是使得程序停留
    getchar();
    //也可以使用
    system("pause");

}
```

![](https://upload-images.jianshu.io/upload_images/1824809-fd40a6fe916fa4cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/633/format/webp)



#### 特别注意的是：

- 程序如果没有最后一句的话，执行完就会退出了。
- 循环的标准C写法：循环变量需要抽取出来，否则在Linux环境下GCC下编译 报错。
- 可以通过sizeof函数来求出某个数据类型所占字节数。
- 可以通过scanf函数来进行输入，第二个参数是变量的地址。





### C语言--指针

学习 C 语言的指针既简单又有趣。通过指针，可以简化一些 C 编程任务的执行，还有一些任务，如动态内存分配，没有指针是无法执行的。所以，想要成为一名优秀的 C 程序员，学习指针是很有必要的。  
正如您所知道的，每一个变量都有一个内存位置，每一个内存位置都定义了可使用连字号（&）运算符访问的地址，它表示了在内存中的一个地址。请看下面的实例，它将输出定义的变量地址：

```c
#include <stdio.h>
 
int main ()
{
   int  var1;
   char var2[10];
 
   printf("var1 变量的地址： %p\n", &var1  );
   printf("var2 变量的地址： %p\n", &var2  );
 
   return 0;
}

结果输出：
var1 变量的地址： 0x7fff5cc109d4
var2 变量的地址： 0x7fff5cc109de
```

### 什么是指针？

指针是一个变量,其值为另一个变量的地址，即，内存位置的直接地址。就像其它变量或常量一样，您必须在使用指针存储其它变量地址之前，对其进行声明。指针变量声明的一般形式为：



```c
type *var-name;
```

在这里，type 是指针的基类型，它必须是一个有效的 C 数据类型，var-name 是指针变量的名称。用来声明指针的星号 * 与乘法中使用的星号是相同的。但是，在这个语句中，星号是用来指定一个变量是指针。以下是有效的指针声明：

```c
int    *ip;    /* 一个整型的指针 */
double *dp;    /* 一个 double 型的指针 */
float  *fp;    /* 一个浮点型的指针 */
char   *ch;     /* 一个字符型的指针 */
```

所有指针的值的实际数据类型，不管是整型、浮点型、字符型，还是其他的数据类型，都是一样的，都是一个代表内存地址的长的十六进制数。不同数据类型的指针之间唯一的不同是，指针所指向的变量或常量的数据类型不同。



### 如何使用指针？

使用指针时会频繁进行以下几个操作：定义一个指针变量、把变量地址赋值给指针、访问指针变量中可用地址的值。这些是通过使用一元运算符 * 来返回位于操作数所指定地址的变量的值。下面的实例涉及到了这些操作：

```c
#include <stdio.h>
#include <stdlib.h>
void main() {
 int var = 20; /* 实际变量的声明 */
 int *ip; /* 指针变量的声明 */
 ip = &var; /* 在指针变量中存储 var 的地址 */
 /* 变量 var 的地址 */
 printf(" 变量 var 的地址: %p\n", &var);
 /* 在指针变量中存储的地址 */
 printf(" 在指针变量中存储的地址: %p\n", ip);
 /* 使用指针访问值 */
 printf(" 指针访问的变量值: %d\n", *ip);
 /*对ip存的地址指向的变量进行操作 */
 *ip = 66;
 printf(" 对ip存的地址指向的变量，即对var进行操作后的值: %d\n", var);
 system("pause");
}
结果输出：
变量 var 的地址: 004FFE3C
在指针变量中存储的地址: 004FFE3C
指针访问的变量值: 20
对ip存的地址指向的变量，即对var进行操作后的值: 66
```



### C 中的 NULL 指针

在变量声明的时候，如果没有确切的地址可以赋值，为指针变量赋一个 NULL 值是一个良好的编程习惯。赋为 NULL 值的指针被称为空指针。  
NULL 指针是一个定义在标准库中的值为零的常量。请看下面的程序：

```c
#include <stdio.h>
 
int main ()
{
   int  *ptr = NULL;
 
   printf("ptr 的值是 %p\n", ptr  );
 
   return 0;
}

结果输出：
ptr 的值是 00000000
```

在大多数的操作系统上，程序不允许访问地址为 0 的内存，因为该内存是操作系统保留的。然而，内存地址 0 有特别重要的意义，它表明该指针不指向一个可访问的内存位置。但按照惯例，如果指针包含空值（零值），则假定它不指向任何东西。  
如需检查一个空指针，您可以使用 if 语句，如下所示：

```c
if(ptr)     /* 如果 p 非空，则完成 */
if(!ptr)    /* 如果 p 为空，则完成 */
```



### 利于指针做简单的游戏外挂(DLL注入方式)

#### 1.游戏程序和外挂程序

先写一个游戏.exe，运行，设定一个游戏时间。

```c
#include <stdlib.h>
#include <stdio.h>
#include <Windows.h>

void main(){
    //游戏时间
    int time = 600;

    //打印出time的地址
    printf("%#x\n",&time);

    while(time>0){
        time--;
        printf("游戏时间剩余%d秒\n",time);
        Sleep(1000);//#include <Windows.h>
    }
    system("pause");
}
```

再创建一个工程（注：生产dll 格式文件），作为外挂，去修改游戏的时间：

```c
#include <stdlib.h>
#include <stdio.h>

__declspec(dllexport) void go(){
    //修改游戏时间
    int* p =(int*)0xdcf8a4;//注意：这个地址是游戏打印出来的
    *p = 99999;
}
```

#### 注意：

- 添加动态库DLL的输出声明：__declspec(dllexport)。
- 要保证外挂程序的地址与游戏程序打印出来的地址一致。
- 把项目的输出改为DLL，而不是EXE。（VS里面：解决方案--属性--常规）。
- 通过DllInject.exe软件把DLL注入到游戏中，就可以发现游戏时间修改为99999了。



#### 2.使用DllInject工具注入外挂程序

首先下载[DllInject.](https://link.jianshu.com/?t=http%3A%2F%2Fwww.pc0359.cn%2Fdowninfo%2F55614.html)

##### （1）点击刷新出现，加载正在运行游戏程序

![](https://upload-images.jianshu.io/upload_images/1824809-40d22f404a18c135.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/489/format/webp)



##### （2）选择目标游戏程序，点击注入按钮，然后注入外挂程序

![](https://upload-images.jianshu.io/upload_images/1824809-30fab2870766d925.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/485/format/webp)



#### 3.注入结果

![](https://upload-images.jianshu.io/upload_images/1824809-a4e2028132c294db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/621/format/webp)

![](https://upload-images.jianshu.io/upload_images/1824809-2b971fb523add6b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620/format/webp)



### 指针类型

##### 指针有类型，地址没有类型

##### 地址只是开始的位置，类型大小决定读取到什么位置结束

示例

```c
#include<stdio.h>
#include<stdlib.h>

void main() {

    int i = 89;
    int *p = &i; //声明int 类型的指针
    double j = 78.9;
    
    printf("int size：%d\n", sizeof(int));
    printf("double size：%d\n", sizeof(double));

    //打印相同类型变量的值
    printf("指针p指向地址 %#x, %d\n", p, *p);

    p = &j;
    printf("指针p指向地址 %#x, %lf\n", p, *p); //想通过4字节读取8字节变量的值，是不行的

    getchar();

}

结果输出：
int size：4
double size：8
指针p指向地址 0xb7f7dc,89
指针p指向地址 0xb7f7c0,0.000000
```

只有“int类型的指针”才能用来指向“int类型的值”；其他不同长度类型的指针不行。  
指针是指向内存种的一块内存空间，而这块空间的大小要根据指针指向的数据的类型的长度来分配。  
比如：int型需要4个字节的空间，double 需要8个字节的空间。  
所以在定义指针的时候要指明指针的类型，这样程序才知道应该在内存中保留多大的空间给这个指针



### 多级指针（主要讨论二级）

#### 多级指针的意义：

##### 动态内存分配二维数组，操作数组的时候。

##### 在jni.h中的struct JNIEnv结构体等有用到。

```c
#include<stdio.h>
#include<stdlib.h>
//多级指针(二级指针)
void main() {

    int a = -50;
    //p1上保存的是a的地址
    int*p1 = &a;

    //p2上保存的是p1的地址
    int** p2 = &p1;

    printf("p1:%#x,p2:%#x\n",p1,p2);

    printf("*p1:%d,**p2:%d\n", *p1, **p2);

    printf("%d\n", a);

    **p2 = 90;

    printf("*p1:%d,**p2:%d\n", *p1, **p2);

    printf(" a:%d\n", a);
    getchar();
}

结果输出：

p1:0xddfc50,p2:0xddfc44
*p1:-50,**p2:-50
-50
*p1:90,**p2:90
 a:90

```



### 指针运算（加减法）（与数组的操作相结合）

#### 指针运算一般在数组的遍历才有意义，基于数组在内存中线性排列方式

```c
#include<stdio.h>
#include<stdlib.h>
void main() {
    int ids[] = { 78,90,23,65,19 };

    //数组的变量名：ids就是数组首地址 3种方式意义一样
    printf("%#x\n", ids);
    printf("%#x\n", &ids);
    printf("%#x\n", &ids[0]);

    //指针变量
    int *p = ids;
    printf("%#x\n", p);

    //p++向前移动sizeof(数据类型)个字节
    p++; 

    //指向数组的下一个元素地址 =  数组首地址值+4;
    printf("%#x\n", p);

    //指向数组的下一个元素
    printf("%d\n", *p);

    getchar();

}

结果输出：
0x11afc7c
0x11afc7c
0x11afc7c
0x11afc7c
0x11afc80
90
```

#### 通过使用指针给数组赋值

```c

void main() {

    int uids[5];
    /*高级写法
    int i = 0;
    for (; i < 5; i++) {
        uids[i] = i;
    }
*/
    //早些版本的写法
    int* p = uids;
    printf("%#x\n", p);
    int i = 0;

    //uids + 5 = uids的首地址值+5*4
    printf("%#x\n", uids + 5);

    for (; p < uids + 5; p++)
    {
        *p = i;
        i++;
    }

    printf("%d\n", uids[0]);
    printf("%d\n", uids[2]);
    getchar();
}

结果输出：
0xf3f888
0xf3f89c
0
2
```



### 函数指针（与Java中的回调类似）

#### 函数指针的定义与基本使用

```c
#include<stdio.h>
#include<stdlib.h>
#include<Windows.h>

void msg(char* msg,char* title) {
        //C语言里面字符串用char指针代替
    MessageBox(0, msg, title,0);

}

void main() {
    printf("%#x\n", msg);
    printf("%#x\n", &msg);

    //函数指针
    void(*fun_p)(char* msg, char* title) = msg;
    fun_p( "消息内容","标题");

    getchar();
}
```

结果输出：  
0x41140  
0x41140

同时弹出window系统提示框

![](https://upload-images.jianshu.io/upload_images/1824809-67364eadc472d772.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/138/format/webp)



#### 在函数指针中调用另一个函数

```c
int add(int a, int b) {
    return a + b;
}

int minus(int a, int b) {
    return a - b;
}

void msg(int(*function_p)(int a, int b), int a, int b) {
    //调用函数指针的函数
    int res = function_p(a, b);
    printf("%d\n", res);
}

void main() {
    msg(add, 1, 2);
    msg(minus, 1, 2);

    getchar();
}

结果输出：
3
-1
```

##### 案例：用随机数生成一个数组，写一个函数查找最小的值，并返回最小数的地址，在主函数中打印出来

```c
int* getMinPointer(int ids[], int len) {
    int i = 0;
    int* p = &ids[0];
    for (; i < len; i++) {
        if (ids[i] < *p) {
            p = &ids[i];
        }
    }
    return p;
}

void main() {
    int ids[10];
    int i = 0;
    //初始化随机数发生器，设置种子，种子不一样，随机数才不一样
    //当前时间作为种子 有符号 int -xx - > +xx
    srand((unsigned)time(NULL));
    for (; i < 10; i++) {
        //100范围内
        ids[i] = rand() % 100;
        printf("%d\n", ids[i]);
    }

    int* p = getMinPointer(ids, sizeof(ids) / sizeof(int));
    printf("%#x,%d\n", p, *p);
    getchar();
}


结果输出：
93
81
23
53
55
49
70
92
65
25
0x10ff8d4,23
```

##### 注意：

- ###### 可以看到，msg函数调用的时候需要传一个函数指针，参数是两个整型，返回值是整型。msg函数的执行需要执行我们手动传入的代码（函数），从而实现了回调（注入代码）。

- ###### 函数指针的使用，Java中new 内部类，类似Java的回调（比回调更加强大）。

- ###### 函数指针，提高复用性，在C语音的回调机制里面非常重要。





### 动态分配内存



### C 内存管理函数

C 语言为内存的分配和管理提供了几个函数。这些函数可以在 <stdlib.h> 头文件中找到。

* void*calloc(int num, int size);  

  在内存中动态地分配 num 个长度为 size 的连续空间，并将每一个字节都初始化为 0。所以它的结果是分配了 num*size 个字节长度的内存空间，并且每个字节的值都是0。

* void free(void *address);  

  该函数释放 address 所指向的内存块,释放的是动态分配的内存空间。

* void *malloc(int num);  

  在堆区分配一块指定大小的内存空间，用来存放数据。这块内存空间在函数执行完成后不会被初始化，它们的值是未知的。

* void *realloc(void *address, int newsize);  

  该函数重新分配内存，把内存扩展到 newsize。



### C语言里面的内存划分

- 栈区（栈内存，存放局部变量，自动分配和释放，里面函数的参数，方法里面的临时变量）
- 堆区（动态内存分配，C语音里面由程序员手动分配），最大值为操作系统的80%
- 全局区或静态区
- 常量区（字符串）
- 程序代码区



### 静态与动态内存分配

在程序运行过程中，动态指定需要使用的内存大小，手动释放，释放之后这些内存还可以被重新使用。

##### 静态内存分配，分配内存大小的是固定，产生的问题：

###### 1.很容易超出栈内存的最大值

###### 2.为了防止内存不够用会开辟更多的内存，容易浪费内存。

##### 动态内存分配，在程序运行过程中，动态指定需要使用的内存大小，手动释放，释放之后这些内存还可以被重新使用



### 栈溢出

下面的代码会导致栈溢出

```c
void main(){
    //属于静态内存分配，分配到栈里面，Window里面每一个应用栈大概是2M，大小确定。与操作系统有关。
    int a [1024 * 1024 * 10 * 4];
}
```

##### 该静态内存定义为40M，而Window里面每一个应用栈大概是2M，超出了范围， 会报stack overflow错误 。



### 动态内存分配与释放

```c
//堆存分配,40M
//参数：字节 KB M 10M 40M
//开辟
int* p1 = (int*)malloc(1024*1024*10*sizeof(int));
//释放
free(p1);
```

### 通过动态内存分配来动态指定数组的大小

在程序运行过长中，可以随意的开辟指定大小的内存，以供使用，相当于Java中的集合

```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>

//创建一个数组，动态指定数组的大小
void main() {
    //静态内存分配创建数组，数组的大小是固定的
    //int i = 10;
    //int a[i];

    int len;

    printf("输入数组的长度：");
    scanf("%d", &len);

    //开辟内存，大小内存len * 4 字节
    int* p = (int*)malloc(len * sizeof(int));//p:数组的首地址

    int i = 0;
    for (; i < len; i++) {
        p[i] = rand() % 100;
        printf("%d,%#x\n", p[i], &p[i]);
    }

    //手动释放内存
    free(p);

    getchar();

}

结果输出：

41,0x513f48
67,0x513f4c
34,0x513f50
0,0x513f54
69,0x513f58
24,0x513f5c
```

### 重新分配realloc

#### 重新分配内存的两种情况：

#### 缩小内存，缩小的那一部分数据会丢失

#### 扩大内存，（连续的）

##### 1.如果当前内存段后面有需要的内存空间，直接扩展这段内存空间，realloc返回原指针

##### 2.如果当前内存段后面的空闲字节不够，那么就使用堆中的第一个能够满足这一要求的内存块，将目前的数据复制到新的位置，并将原来的数据库释放掉，返回新的内存地址

##### 3.如果申请失败，返回NULL，原来的指针仍然有效

```c

void main() {
    int len;
    printf("第一次输入数组的长度：");
    scanf("%d", &len);

    //开辟内存，大小内存len * 4 字节
    int* p = (int*)malloc(len * sizeof(int));//p:数组的首地址

    int i = 0;
    for (; i < len; i++) {
        p[i] = rand() % 100;
        printf("%d,%#x\n", p[i], &p[i]);
    }


    int addLen;
    printf("输入数组增加的长度:");
    scanf("%d", &addLen);


    int* p2 = (int*)realloc(p, sizeof(int) * (len + addLen));
    if (p2 == NULL) {
        printf("重新分配失败......");
    }

    printf("------------新数组-------------------\n");
    //重新赋值
    i = 0;
    for (; i < len + addLen; i++) {
        p2[i] = rand() % 200;
        printf("%d,%#x\n", p2[i], &p2[i]);
    }

    //手动释放内存 p2释放内存 p也会释放，因为给p2分配内存的时候要么p已经释放，要么p2、p指向统一地址区域
    if (p2 != NULL) {
        free(p2);
        p2 = NULL;
    }
    getchar();

}

结果输出：
第一次输入数组的长度：5
41,0x5e4ad8
67,0x5e4adc
34,0x5e4ae0
0,0x5e4ae4
69,0x5e4ae8
输入数组增加的长度:5
------------新数组-------------------
124,0x5e4ad8
78,0x5e4adc
158,0x5e4ae0
162,0x5e4ae4
64,0x5e4ae8
105,0x5e4aec
145,0x5e4af0
81,0x5e4af4
27,0x5e4af8
161,0x5e4afc
```

#### 内存分配的几个注意细节

##### 1.不能多次释放

##### 2.释放完之后（指针仍然有值），给指针置NULL，标志释放完成

##### 3.内存泄露（p重新赋值之后，再free，并没有真正释放内存）

#### 避免内存泄漏

##### p重新赋值之前先free

内存泄漏写法：

```c
void main(){
    //40M
    int* p1 = malloc(1024 * 1024 * 10 * sizeof(int));
    //free(p1);
    //p1 = NULL;
    //错误，没有立即释放内存
    printf("%#x\n",p1);

    //80M
    p1 = malloc(1024 * 1024 * 10 * sizeof(int) * 2);
    
    free(p1);
    p1 = NULL;

    getchar();
}
```

打开任务管理器，看到有40M内存泄漏。

正确写法：

```c
void main(){
    //40M
    int* p1 = malloc(1024 * 1024 * 10 * sizeof(int));
    free(p1);
    p1 = NULL;
    printf("%#x\n",p1);

    //80M
    p1 = malloc(1024 * 1024 * 10 * sizeof(int) * 2);
    
    free(p1);
    p1 = NULL;

    getchar();
}
```

打开任务管理器，看到内存只有0.3M，正常。






