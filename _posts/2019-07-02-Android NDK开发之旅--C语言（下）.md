---
layout:     post
title:      "Android NDK开发之旅-C语言（下）"
subtitle:   ""
date:       2019-07-02 10:37:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---


##### 共用体是一种特殊的数据类型，允许您在相同的内存位置存储不同的数据类型。您可以定义一个带有多成员的共用体，但是任何时候只能有一个成员带有值。共用体提供了一种使用相同的内存位置的有效方式。



#### 定义共用体

为了定义共用体，您必须使用 union 语句，方式与定义结构类似。union 语句定义了一个新的数据类型，带有多个成员。union 语句的格式如下：

```c
union [union tag]
{
   member definition;
   member definition;
   ...
   member definition;
} [one or more union variables];
```

union tag 是可选的，每个 member definition 是标准的变量定义，比如 int i; 或者 float f; 或者其他有效的变量定义。在共用体定义的末尾，最后一个分号之前，您可以指定一个或多个共用体变量，这是可选的。下面定义一个名为 Data 的共用体类型，有三个成员 i、f 和 str：

```c
union Data
{
   int i;
   float f;
   char  str[20];
} data;
```

现在，Data 类型的变量可以存储一个整数、一个浮点数，或者一个字符串。这意味着一个变量（相同的内存位置）可以存储多个多种类型的数据。您可以根据需要在一个共用体内使用任何内置的或者用户自定义的数据类型。



#### 共用体占用的内存应足够存储共用体中最大的成员。

例如，在上面的实例中，Data 将占用 20 个字节的内存空间，因为在各个成员中，字符串所占用的空间是最大的。下面的实例将显示上面的共用体占用的总内存大小：

```c
union Data
{
   int i;
   float f;
   char  str[20];
};
 
void main( )
{
   union Data data;        
 
   printf( "Memory size occupied by data : %d\n", sizeof(data));
 
   system("pause");
}

结果输出：

Memory size occupied by data : 20
```

#### 联合变量任何时刻只有一个变量存在，最后一次赋值有效

```c
union  MyValue {

    int x;
    int y;
    double z;

};



void main() {

    union MyValue d1;

    d1.x = 90;

    d1.y = 100; //最后一次赋值有效

    //d1.z = 23.8;

    printf("%d , %d, %lf\n", d1.x, d1.y, d1.z);

    d1.z = 23.8;
    printf("%d, %d, %lf\n", d1.x, d1.y, d1.z);

    system("pause");

}

结果输出：

100 , 100, -92559592117433135502616407313071917486139351398276445610442752.000000
-858993459, -858993459, 23.800000
```

#### JNI头文件中的联合体：

```c
typedef union jvalue {
    jboolean    z;
    jbyte       b;
    jchar       c;
    jshort      s;
    jint        i;
    jlong       j;
    jfloat      f;
    jdouble     d;
    jobject     l;
} jvalue;
```

### 枚举

##### 枚举（列举所有的情况），限定值的取值范围，保证取值的安全性。

```c
enum Day
{
    Monday,//默认为0，后续枚举成员的值在前一个成员上加1
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
    Sunday
};


void main() {
    //枚举的值，必须是括号中的值
    enum Day d = Monday;
    printf("%#x,%d\n", &d, d);

     d = Wednesday;
    printf("%#x,%d\n", &d, d);

    getchar();
}
结果输出：

0xdaaff5e4,0
0xdaaff5e4,2
```

(1) 枚举型是一个集合，集合中的元素(枚举成员)是一些命名的整型常量，元素之间用逗号,隔开。

(2) DAY是一个标识符，可以看成这个集合的名字，是一个可选项，即是可有可无的项。

(3) 第一个枚举成员的默认值为整型的0，后续枚举成员的值在前一个成员上加1。

(4) 可以人为设定枚举成员的值，从而自定义某个范围内的整数。

(5) 枚举型是预处理指令#define的替代。

(6) 类型定义以分号;结束。

综合举例

```c
enum Season
{
    spring, summer = 100, fall = 96, winter
};

typedef enum
{
    Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday
}
Weekday;

void main()
{
    /* Season */
    printf("%d \n", spring); // 0
    printf("%d, %c \n", summer, summer); // 100, d
    printf("%d \n", fall + winter); // 193
    enum Season mySeason = winter;
    if (winter == mySeason)
        printf("mySeason is winter \n"); // mySeason is winter

    int x = 100;
    if (x == summer)
        printf("x is equal to summer\n"); // x is equal to summer

    printf("%d bytes\n", sizeof(spring)); // 4 bytes

                                          /* Weekday */
    printf("sizeof Weekday is: %d \n", sizeof(Weekday)); //sizeof Weekday is: 4

    Weekday today = Saturday;
    Weekday tomorrow;
    if (today == Monday)
        tomorrow = Tuesday;
    else
        tomorrow = (Weekday)(today + 1); //remember to convert from int to Weekday


    getchar();
}

结果输出：

0
100, d
193
mySeason is winter
x is equal to summer
4 bytes
```

### 文件读写

一个文件，无论它是文本文件还是二进制文件，都是代表了一系列的字节。C 语言不仅提供了访问顶层的函数，也提供了底层（OS）调用来处理存储设备上的文件。

#### 打开文件

您可以使用 fopen( ) 函数来创建一个新的文件或者打开一个已有的文件，这个调用会初始化类型 FILE 的一个对象，类型 FILE 包含了所有用来控制流的必要的信息。下面是这个函数调用的原型：

```c
FILE *fopen( const char * filename, const char * mode );
```

在这里，filename 是字符串，用来命名文件，访问模式 mode 的值可以是下列值中的一个：

* r 打开一个已有的文本文件，允许读取文件。

* w 打开一个文本文件，允许写入文件。如果文件不存在，则会创建一个新文件。在这里，您的程序会从文件的开头写入内容。

*  a 打开一个文本文件，以追加模式写入文件。如果文件不存在，则会创建一个新文件。在这里，您的程序会在已有的文件内容中追加内容。

*  r+ 打开一个文本文件，允许读写文件。

*  w+ 打开一个文本文件，允许读写文件。如果文件已存在，则文件会被截断为零长度，如果文件不存在，则会创建一个新文件。

*  a+ 打开一个文本文件，允许读写文件。如果文件不存在，则会创建一个新文件。读取会从文件的开头开始，写入则只能是追加模式。

如果处理的是二进制文件，则需使用下面的访问模式来取代上面的访问模式：

```c
"rb", "wb", "ab", "rb+", "r+b", "wb+", "w+b", "ab+", "a+b"
```

#### 关闭文件

为了关闭文件，请使用 fclose( ) 函数。函数的原型如下：

```c
 int fclose( FILE *fp );
```

如果成功关闭文件，fclose( ) 函数返回零，如果关闭文件时发生错误，函数返回 EOF。这个函数实际上，会清空缓冲区中的数据，关闭文件，并释放用于该文件的所有内存。EOF 是一个定义在头文件 stdio.h 中的常量。  
C 标准库提供了各种函数来按字符或者以固定长度字符串的形式读写文件。

#### 读取文件

下面是从文件读取单个字符的最简单的函数：

```c
int fgetc( FILE * fp );
```

fgetc() 函数从 fp 所指向的输入文件中读取一个字符。返回值是读取的字符，如果发生错误则返回 EOF。下面的函数允许您从流中读取一个字符串：

```c
char *fgets( char *buf, int n, FILE *fp );
```

函数 fgets() 从 fp 所指向的输入流中读取 n - 1 个字符。它会把读取的字符串复制到缓冲区 buf，并在最后追加一个 null 字符来终止字符串。  
如果这个函数在读取最后一个字符之前就遇到一个换行符 '\n' 或文件的末尾 EOF，则只会返回读取到的字符，包括换行符。您也可以使用 int fscanf(FILE *fp, const char *format, ...) 函数来从文件中读取字符串，但是在遇到第一个空格字符时，它会停止读取。

示例

```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>

//读取文本文件
void main() {

    char path[] = "C:\\Users\\Administrator\\Desktop\\friend.txt"; //文本中语句为 hello world

    //打开
    FILE *fp = fopen(path, "r");
    if (fp == NULL)
    {
        printf("文件打开失败...");
        return;
    }
    //读取
    char buff[50];//缓冲
    while (fgets(buff,50,fp)) {
        printf("%s", buff);
    }

    //关闭
    fclose(fp);

    getchar();

}

结果输出：

 hello world
```

#### 写入文件

下面是把字符写入到流中的最简单的函数：

```c
int fputc( int c, FILE *fp );
```

函数 fputc() 把参数 c 的字符值写入到 fp 所指向的输出流中。如果写入成功，它会返回写入的字符，如果发生错误，则会返回 EOF。您可以使用下面的函数来把一个以 null 结尾的字符串写入到流中：

```c
int fputs( const char *s, FILE *fp );
```

函数 fputs() 把字符串 s 写入到 fp 所指向的输出流中。如果写入成功，它会返回一个非负值，如果发生错误，则会返回 EOF。您也可以使用 int fprintf(FILE *fp,const char *format, ...) 函数来写把一个字符串写入到文件中。尝试下面的实例：

```c
//写入文本文件
void main() {
    char *path = "C:\\Users\\Administrator\\Desktop\\test.txt";
    //打开
    FILE *fp = fopen(path, "w");
    char *text = "今天天气不错\n出去玩吧!";
    fputs(text,fp);

    //关闭
    fclose(fp);
    getchar();

}


在test文本中输出：
今天天气不错
出去玩吧!
```

##### 注意：请确保您有可用的 /tmp 目录，如果不存在该目录，则需要在您的计算机上先创建该目录。

#### 读写二进制I/O文件

计算机的文件存储在物理上都是二进制，文本文件和二进制之分，其实是一个人为的逻辑之分。

C读写文本文件与二进制文件的差别仅仅体现在回车换行符：

1.写文本时，每遇到一个'\n'，会将其转换成'\r\n'(回车换行)。  
2.读文本时，每遇到一个'\r\n'，会将其转换成'\n'。  
3.但是读写二进制文件的时候并不会做以上转换。



##### 函数原型：

```c
size_t fread ( void * ptr, size_t size, size_t count, FILE * stream );
```

其中，ptr：指向保存结果的指针；size：每个数据类型的大小；count：数据的个数；stream：文件指针  
函数返回读取数据的个数。

```c
size_t fwrite ( const void * ptr, size_t size, size_t count, FILE * stream );
```

其中，ptr：指向保存数据的指针；size：每个数据类型的大小；count：数据的个数；stream：文件指针  
函数返回写入数据的个数。

下面是二进制文件读写的例子（图片的复制）：

```c
void main() {
    char *read_path = "D:\\BaiduNetdiskDownload\\ndk\\2016_08_08_C_联合体_枚举_IO\\files\\girl.png";
    char *write_path = "D:\\BaiduNetdiskDownload\\ndk\\2016_08_08_C_联合体_枚举_IO\\files\\girl_new.png";
    //b字符表示操作二进制文件binary
    FILE *read_fp  = fopen(read_path, "rb");
    //写的文件
    FILE *write_fp = fopen(write_path,"wb");

    //复制
    int buff[50]; //缓冲区域
    int len = 0;//每次读到的数据长度  
    while ((len = fread(buff,sizeof(int), 50,read_fp))!=0) {//50 是写的比较大的一个数
        //将读到的内容写入新的文件
        fwrite(buff, sizeof(int), len, write_fp);
    }

    fclose(read_fp);
    fclose(write_fp);

    getchar();

}
```

#### 获取文件的大小

```c
void main() {
    char *read_path = "D:\\BaiduNetdiskDownload\\ndk\\2016_08_08_C_联合体_枚举_IO\\files\\girl.png";
    FILE *fp = fopen(read_path, "r");
    //重新定位文件指针
    //SEEK_END文件末尾，0偏移量
    fseek(fp, 0, SEEK_END);
    //返回当前的文件指针，相对于文件开头的位移量
    long filesize = ftell(fp);
    printf("%d\n", filesize);

    getchar();
}
```

#### 文本简单加密、解密

```c
void crypt(char normal_path[], char crypt_path[]) {
    //打开文件
    FILE *normal_fp = fopen(normal_path, "r");
    FILE *crypt_fp = fopen(crypt_path, "w");
    //一次读取一个字符
    int ch;
    while ((ch = fgetc(normal_fp)) != EOF) { //End of File
        //写入（异或运算）
        fputc(ch ^ 9, crypt_fp);
    }
    //  关闭
    fclose(crypt_fp);
    fclose(normal_fp);
}

//解密
void decrypt(char crypt_path[],char decrypt_path[]) {
    //打开文件
    FILE *normal_fp = fopen(crypt_path,"r");
    FILE *crypt_fp = fopen(decrypt_path, "w");
    //一次读取一个字符
    int ch;
    while ((ch = fgetc(normal_fp)) !=EOF)//End of File
    {
        //写入(异或运算)
        fputc(ch ^ 9, crypt_fp);
    }
    //关闭
    fclose(crypt_fp);
    fclose(normal_fp);

}

void main() {
    char *normal_path = "D:\\userinfo.txt";
    char *crypt_path = "D:\\userinfo_crypt.txt";
    char *decrypt_path = "D:\\userinfo_decrypt.txt";
    //加密文件
    crypt(normal_path, crypt_path);
    //解密文件
    decrypt(crypt_path, decrypt_path);

    getchar();
}
```

#### 二进制文件简单加解密

```c
void crypt(char normal_path[], char crypt_path[], char password[]) {
    //打开文件
    FILE *normal_fp = fopen(normal_path, "rb");
    FILE *crypt_fp = fopen(crypt_path, "wb");
    //一次读取一个字符
    int ch;
    int i = 0; //循环使用密码中的字母进行异或运算
    int pwd_len = strlen(password); //密码的长度
    while ((ch = fgetc(normal_fp)) != EOF) { //End of File
    //写入（异或运算）
        fputc(ch ^ password[i % pwd_len], crypt_fp);
        i++;
    }
    //关闭
    fclose(crypt_fp);
    fclose(normal_fp);
}

//解密
void decrypt(char crypt_path[], char decrypt_path[], char password[]) {
    //打开文件
    FILE *normal_fp = fopen(crypt_path, "rb");
    FILE *crypt_fp = fopen(decrypt_path, "wb");
    //一次读取一个字符
    int ch;
    int i = 0; //循环使用密码中的字母进行异或运算
    int pwd_len = strlen(password); //密码的长度
    while ((ch = fgetc(normal_fp)) != EOF) { //End of File
    //写入（异或运算）
        fputc(ch ^ password[i % pwd_len], crypt_fp);
        i++;
    }
    //关闭
    fclose(crypt_fp);
    fclose(normal_fp);

}

void main() {
    char *normal_path = "D:\\girl.png";
    char *crypt_path = "D:\\girl_crypt.png";
    char *decrypt_path = "D:\\girl_decrypt.png";

    //加密文件
    crypt(normal_path, crypt_path, "123456");

    //加密文件
    decrypt(crypt_path, decrypt_path, "123456");

    getchar();
}
```

##### 一般腾讯、阿里等大公司的用户关键数据是用C\C++（动态库so反编译很难）加密的。因为Java的加密方法反编译比较容易破解。



### 预编译

C执行过程  
1、编译：形成目标代码  
2、链接：将目标代码与C的函数库相链接，合并代码，生成可执行文件。  
3、运行



###### 示例：

Test.txt

```c
printf("I love coding\n");
```

main.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

 int main(){
     //预编译:为了编译做准备，将文件的完整代码直接拷贝过来（替换）
     //在编译之前,做一些事情
     //#include "Test.txt" 等价于 printf("I have a Dream.");
     #include "Test.txt"
     getchar();
     return 0;
 }

结果输出：
I love coding
```

### C 预处理器

C 预处理器不是编译器的组成部分，但是它是编译过程中一个单独的步骤。简言之，C 预处理器只不过是一个文本替换工具而已，它们会指示编译器在实际编译之前完成所需的预处理。我们将把 C 预处理器（C Preprocessor）简写为 CPP。  
所有的预处理器命令都是以井号（#）开头。它必须是第一个非空字符，为了增强可读性，预处理器指令应从第一列开始。下面列出了所有重要的预处理器指令：



#### 宏定义、宏替换

##### 1、定义标识

###### (1)、例如通过判断一些标识是否定义来判断是否支持某种语法、平台等等：

//表示支持C++语法

```c
#ifdef __cplusplus

#endif // __cplusplus
```

//表示支持Android、Windows、苹果平台等等

```c
#ifdef ANDROID

#endif // ANDROID
```

###### (2)、防止头文件重复引入：

举个例子，我们有三个文件a.h、b.h、Test.cpp，分别如下：

这是a.h：

```c
#include "b.h"

void a();
```

这是b.h：

```c
#include "a.h"

void b();
```

最后Test.cpp里面引用了a.h

```c
#include "a.h"
```

这样，当Test包含a的时候，a又会去包含b，b又会包含a，这样就会造成循环包含。类似于Hibernate里面的SQL循环引用。最终会报如下错误：

```c
fatal error C1014: 包含文件太多 : 深度 = 1024
```

通过宏定义判断就可以解决这个问题：  
A.h

```c
#ifndef AH //如果没有定AH 就只执行后面语句 ，如果定义就不执行后面语句防止多次include ，引入头文件
#define AH
#include "B.h"
void printfA();

#endif
```

B.h

```c
#ifndef BH
#define BH
#include "A.h"
void printfB();

#endif
```

另外，新版本可通过#pragma once语句解决这个问题。

```c
//该头文件只被包含一次，编译器自动处理循环包含问题
#pragma once
#include "B.h"
```

##### 2.定义常数(便于修改和阅读)

```c
#define MAX 100

void main(){
    int i = 100;
    if (i == MAX){
        printf("哈哈");
    }
    getchar();
}
```

##### 3、定义“宏函数”。

实质上就是一个替换的过程。

###### 示例：

```c
#include <stdio.h>
#include <stdlib.h>

#define GET_MIN(A,B) A < B ? A : B

 int main(){
     int a = 100;
     int b = 200;
     int c = GET_MIN(100, 200);
     printf("最小值: %d\n", c);
     getchar();
     return 0;
 }

结果输出：
最小值: 100
```

###### Android中LOG简化示例：

C语言宏函数打印语句简单使用：

```
#define LOG(FORMAT,...) printf(FORMAT,__VA_ARGS__);
```

`...` 三个点表示可变参数与`__VA_ARGS__` 对应  
LOG会有级别，于是进一步升级：

```
#define LOG_I(FORMAT,...) printf("INFO:"); printf(##FORMAT,__VA_ARGS__);
#define LOG_E(FORMAT,...) printf("ERROR:"); printf(##FORMAT,__VA_ARGS__);
```

进一步简化重复代码，重复LEVEL日志级别：

```c
#define LOG(LEVEL,FORMAT,...) printf(LEVEL);printf(##FORMAT,__VA_ARGS__);
#define LOG_I(FORMAT,...)LOG("INFO:",FORMAT,__VA_ARGS__);
#define LOG_E(FORMAT,...)LOG("ERROR:",FORMAT,__VA_ARGS__);
#define LOG_D(FORMAT,...)LOG("DEBUG:",FORMAT,__VA_ARGS__);
```

在Android JNI开发的时候，我们打印一句日志是通过__android_log_print函数来实现的

```
__android_log_print(ANDROID_LOG_INFO, "FFmpeg", "%s", "fix");
```

同理，实际使用通过宏定义简化代码：

```
#define LOGI(FORMAT,...) __android_log_print(ANDROID_LOG_INFO,"FFmpeg",FORMAT,##__VA_ARGS__);
LOGI("%s","fix");
```

#### 预定义宏

ANSI C 定义了许多宏。在编程中您可以使用这些宏，但是不能直接修改这些预定义的宏。

| __ DAT__  | 当前日期，一个以 "MMM DD YYYY" 格式表示的字符常量。 |
| --------- | --------------------------------- |
| __ TIME__ | 当前时间，一个以 "HH:MM:SS" 格式表示的字符常量。    |
| __ FILE__ | 这会包含当前文件名，一个字符串常量。                |
| __ LINE__ | 这会包含当前行号，一个十进制常量。                 |
| __ STDC__ | 当编译器以 ANSI 标准编译时，则定义为 1。          |

```c
#include <stdio.h>

main()
{
   printf("File :%s\n", __FILE__ );
   printf("Date :%s\n", __DATE__ );
   printf("Time :%s\n", __TIME__ );
   printf("Line :%d\n", __LINE__ );
   printf("ANSI :%d\n", __STDC__ );

}
```

当上面的代码（在文件 test.c 中）被编译和执行时，它会产生下列结果：

```
File :test.c
Date :Jun 2 2012
Time :03:36:24
Line :8
ANSI :1
```

#### 预处理器运算符

C 预处理器提供了下列的运算符来帮助您创建宏：  
宏延续运算符（\）  
一个宏通常写在一个单行上。但是如果宏太长，一个单行容纳不下，则使用宏延续运算符（\）。例如：

```c
#define  message_for(a, b)  \
    printf(#a " and " #b ": We love you!\n")
```

##### 字符串常量化运算符`#`

###### 在宏定义中，当需要把一个宏的参数转换为字符串常量时，则使用字符串常量化运算符`#`。在宏中使用的该运算符有一个特定的参数或参数列表。例如：

```c
#include <stdio.h>

#define  message_for(a, b)  \
    printf(#a " and " #b ": We love you!\n")

int main(void)
{
   message_for(Carole, Debra);
   return 0;
}
```

当上面的代码被编译和执行时，它会产生下列结果：

```
Carole and Debra: We love you!
```

##### 标记粘贴运算符`##`

###### 宏定义内的标记粘贴运算符`##`会合并两个参数。它允许在宏定义中两个独立的标记被合并为一个标记。

###### 示例1

```c
#include <stdio.h>

#define tokenpaster(n) printf ("token" #n " = %d", token##n)

int main(void)
{
   int token34 = 40;

   tokenpaster(34);
   return 0;
}

结果输出：
token34 = 40
```

因为这个示例会从编译器产生下列的实际输出：

```
printf ("token34 = %d", token34);
```

这个示例演示了 `token##n` 会连接到`token34` 中，在这里，我们使用了字符串常量化运算符`#`和标记粘贴运算符`##`。

###### 示例2

```
 #include <stdio.h>
 //方法名很长(方法名称有规律)
  int com_haocai_ndk_get_min(int a,int b){
      return a < b ? a : b;
  }

  int com_haocai_ndk_get_max(int a,int b){
      return a > b ? a : b;
  }
  //语法规范
  // #define 标示名(方法名,A,B) com_tz_ndk_get_##NAME(A,B)
  #define call(NAME,A,B) com_haocai_ndk_get_##NAME(A,B)

  int main(){
      int c = call(min,100,200);
      printf("最小值:%d\n",c);
      int d = call(max,100,200);
      printf("最大值:%d\n",d);
      getchar();
      return 0;
  }
结果输出：
最小值:100
最大值:200
```

##### defined() 运算符

预处理器 `defined` 运算符是用在常量表达式中的，用来确定一个标识符是否已经使用 `#define`定义过。如果指定的标识符已定义，则值为真（非零）。如果指定的标识符未定义，则值为假（零）。下面的实例演示了 `defined()` 运算符的用法：

```
#include <stdio.h>

#if !defined (MESSAGE)
   #define MESSAGE "You wish!"
#endif

int main(void)
{
   printf("Here is the message: %s\n", MESSAGE);  
   return 0;
}
```

当上面的代码被编译和执行时，它会产生下列结果：

```
Here is the message: You wish!
```

##### 参数化的宏

CPP 一个强大的功能是可以使用参数化的宏来模拟函数。例如，下面的代码是计算一个数的平方：

```
int square(int x) {
   return x * x;
}
```

我们可以使用宏重写上面的代码，如下：

```
#define square(x) ((x) * (x))
```

在使用带有参数的宏之前，必须使用 #define 指令定义。参数列表是括在圆括号内，且必须紧跟在宏名称的后边。宏名称和左圆括号之间不允许有空格。例如：

```
#include <stdio.h>

#define MAX(x,y) ((x) > (y) ? (x) : (y))

int main(void)
{
   printf("Max between 20 and 10 is %d\n", MAX(10, 20));  
   return 0;
}.
```

当上面的代码被编译和执行时，它会产生下列结果：

```
Max between 20 and 10 is 20
```






