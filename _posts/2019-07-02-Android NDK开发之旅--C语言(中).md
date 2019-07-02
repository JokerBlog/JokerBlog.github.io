---
layout:     post
title:      "Android NDK开发之旅--C语言(中)"
subtitle:   ""
date:       2019-07-01 16:56:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### C语音的字符串有两种：

#### 字符数组

##### 数组可以修改其中某一个值，不可以整体赋值。

```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>

//使用字符数组存储字符串
void  main() {

    //三种写法
    //\0代表 结束符
    char str[] = { 'h','e','l','l','o','\0' };
    //char str[6]= { 'h','e','l','l','o' };
    //char str[10] = "hello";

    printf("%s\n", str);
    //地址
    printf("%#x\n", str);
    getchar();
}

结果输出：

hello
0xb3fb78
```

#### 字符指针

##### 字符指针不可以修改其中某一个值，可以整体赋值。使用指针加法，结合结束符，可以进行截取。

```c
void main() {
    char *str = "how are you？";
    printf("%s\n", str);
    //str[1] = "w" ; //字符指针不可以修改其中某一个值

    str = "hello world";
    printf("%s\n", str);
    printf("%#x\n", str);

    //使用指针加法，截取字符串
    str += 3; //指向第四个字符首地址
    while (*str) {
        printf("%c", *str);
        str++;

    }
    getchar();
}
结果输出：

how are you？
hello world
0x97b44
lo world
```

### 字符串常用的方法

在线手册：[http://www.kuqin.com/clib/](https://link.jianshu.com?t=http://www.kuqin.com/clib/)  
相关的头文件：#include <string.h>

#### strcpy字符串拼接

原型：extern char *strcpy(char *dest,char *src);

功能：把src所指由NULL结束的字符串复制到dest所指的数组中。

说明：src和dest所指内存区域不可以重叠且dest必须有足够的空间来容纳src的字符串。  
返回指向dest的指针。

举例：

```c
void main(void){
    char dest[50];  
    char *a = "china";
    char *b = " is powerful!";
    strcpy(dest, a);
    strcat(dest, b);
    printf("%s\n", dest);

    system("pause");
}
结果输出：

china is powerful!
```

#### strchr字符串中查找字符

原型：extern char *strchr(char *s,char c);

功能：查找字符串s中首次出现字符c的位置

说明：返回首次出现c的位置的指针，如果s中不存在c则返回NULL。

```c
void main(void){
    char *str = "I want go to USA!";
    printf("%#x\n", str);
    //U元素的指针
    //str+3
    char* p = strchr(str, 'w');
    if (p){
        printf("索引位置：%d\n", p - str);
    }
    else{
        printf("没有找到");
    }

    system("pause");
}

结果输出：

0x877b30
索引位置：2
```



### 更多用法

```c
//strset 把字符串s中的所有字符都设置成字符c
void main(void){
    char str[] = "internet change the world!";
    _strset(str,'w');
    printf("%s\n",str);
    system("pause");
}

//strrev 把字符串s的所有字符的顺序颠倒过来
void main(void){
    char str[] = "internet change the world!";
    _strrev(str);
    printf("%s\n", str);
    system("pause");
}

//atoi 字符串转为int类型
//atol()：将字符串转换为长整型值
void main(void){
    char* str = "a78";
    //int r = atoi(str);    
    printf("%d\n", r);

    system("pause");
}

// 字符串转为double类型
void main(void){
    char* str = "77b8b";
    char** p = NULL;
    //char* p = str + 2;
    //参数说明：str为要转换的字符串，endstr 为第一个不能转换的字符的指针
    double r = strtod(str,p);
    printf("%lf\n", r);
    printf("%#x\n", p);

    system("pause");
}

//strupr转换为大写
void main(void){
    char str[] = "CHINA motherland!";
    _strupr(str);
    printf("%s\n",str);
    system("pause");
}

//转换为小写
void mystrlwr(char str[],int len){
    int i = 0;
    for (; i < len; i++){
        //A-Z 字母 a-Z
        if (str[i] >= 'A' && str[i] <= 'Z'){
            str[i] = str[i]-'A' + 'a';
        }
    }   

}


void main(void){
    char str[] = "CHINA motherland!";
    mystrlwr(str,strlen(str));
    printf("%s\n", str);
    system("pause");
}

//练习：删除字符串中指定的字符
void delchar(char *str, char del){
    char *p = str;
    while (*str != '\0') {
        if (*str != del) {
            *p++ = *str;
        }
        str++;
    }
    *p = '\0';
}


//删除最后一个字符
int main()
{
    char str[] = "vencent ppqq";

    delchar(str,'t');
    printf("%s\n", str);
    
    system("pause");
}

//Java String replaceAll 
//StringBuffer buff.deleteCharAt(buff.length()-1);
//删除最后一个字符
void main(void){
    char str[] = "internet,";
    str[strlen(str) - 1] = '\0';
    printf("%s\n", str);
        
    //作业：realloc实现StringBuffer的拼接，而不是一开始开辟一个很大的数组
    //结构体StringBuffer 

    system("pause");
}

//memcpy 由src所指内存区域复制count个字节到dest所指内存区域
void main(void){
    char src[] = "C,C++,Java";
    char dest[20] = {0};

    //字节
    memcpy(dest,src,5);
    
    printf("%s\n",dest);
    system("pause");
}

//memchr 从buf所指内存区域的前count个字节查找字符ch。
void main(void){
    char src[] = "C,C++,Java";
    char ch = 'C';

    //字节 (分段截取)
    char* p = memchr(src+3, ch, 5);
    if (p){
        printf("索引：%d\n", p - src);
    }
    else{
        printf("找不到\n");
    }

    
    system("pause");
}

//memmove 由src所指内存区域复制count个字节到dest所指内存区域。
void main(){
    char s[] = "Michael Jackson!";
    //截取的效果 
    memmove(s, s + 8, strlen(s) - 8 - 1);
    s[strlen(s) - 8] = 0;
    printf("%s\n", s);
    getchar();
}

//在字符串s1中寻找字符串s2中任何一个字符相匹配的第一个字符的位置，空字符NULL不包括在内
void main(){
    char *s1 = "Welcome To Beijing";
    char *s2 = "to"; 
    char *p;

    p = strpbrk(s1, s2);
    if (p)
        printf("%s\n", p);
    else
        printf("Not Found!\n");

    p = strpbrk(s1, "Da");
    if (p)
        printf("%s", p);
    else
        printf("Not Found!");

    getchar();
}
```

### 结构体

### 概念、定义于初始化方式

##### C 数组允许定义可存储相同类型数据项的变量，结构是 C 编程中另一种用户自定义的可用的数据类型，它允许您存储不同类型的数据项。

结构用于表示一条记录，假设您想要跟踪图书馆中书本的动态，您可能需要跟踪每本书的下列属性：

- Title
- Author
- Subject
- Book ID



### 定义结构

为了定义结构，您必须使用 struct 语句。struct 语句定义了一个包含多个成员的新的数据类型，struct 语句的格式如下

```c
struct [structure tag]
{
   member definition;
   member definition;
   ...
   member definition;
} [one or more structure variables];
```

structure tag 是可选的，每个 member definition 是标准的变量定义，比如 int i; 或者 float f; 或者其他有效的变量定义。在结构定义的末尾，最后一个分号之前，您可以指定一个或多个结构变量，这是可选的。下面是声明 Book 结构的方式：

```c
struct Books
{
   char  title[50];
   char  author[50];
   char  subject[100];
   int   book_id;
} book;
```

### 简单示例

```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <Windows.h>
#include <string.h>


//结构体是一种构造数据类型
//把不同的数据类型整合起来成为一个自定义的数据类型

struct Man {
    //成员
    char name[20];
    int age;


};
void main() {
    //初始化结构体的变量
    //第一种方式
    struct Man m1 = {"Jack",21};
    printf("%s，%d \n", m1.name, m1.age);
    //第二种方式
    struct Man m2;
    
    m2.age = 23;
    //m2.name = "rose"; 不能这样赋值
    strcpy(m2.name, "rose");
    //或者
    //sprintf(m2.name, "Jason");


    //不能再赋值
    //m1 = {};//类似JavaScript字面量赋值，只能在变量声明时赋值

    printf("%s，%d", m2.name, m2.age);


    getchar();

}

结果输出：
Jack，21
rose，23
```

#### 结构体可以在定义之后跟着声明或者初始化变量

```c
struct Man {
    char name[20];
    int age;
}m1, m2 = {"Lucy",35};  //m1结构体变量名

void main() {
    strcpy(m1.name,"Jack");
    m1.age = 10;

    printf("%s，%d \n", m1.name, m1.age);
    printf("%s，%d \n", m2.name, m2.age);
    getchar();
}

结果输出：
Jack，10
Lucy，35
```

#### 匿名结构体

控制结构体变量的个数（限量版），相当于单例

```c
struct {
    char name[20];
    int age;

}m1;
```

#### 结构体嵌套

```c
struct Teacher
{
    char name[20];
};

struct Student 
{
    char name[20];
    int age;
    struct Teacher t;
};

void main() {
    //字面量的方式
    struct Student s1 = { "jack" ,21,{"Jeason"} };
    printf("%s，%d, %s\n", s1.name, s1.age, s1.t.name);
    system("pause");
}
结果输出：
jack，21, Jeason

```



#### 结构体与指针

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

#### 指针与结构体数组

```c
struct Man
{
    char name[20];
    int  age;
};

void main() {
    struct Man mans[] = { {"Jack",20},{"Rose",19} };

    //遍历数组
    //第一种方式
    struct Man *p = mans;
    for (; p < mans + 2;p++) {
        printf("%s, %d\n", p->name, p->age);
    }
    //第二种方式
    int i = 0;
    for (; i <   sizeof(mans)/sizeof(struct Man); i++)
    {
        printf("%s, %d\n", mans[i].name, mans[i].age);
    }

    system("pause");
}

结果输出：
Jack, 20
Rose, 19
Jack, 20
Rose, 19
```

#### 结构体的大小

##### 字节对齐，结构体变量的大小，必须是最宽基本数据类型的整数倍。通过空间换取时间来提升读取效率。

##### 宽基本数据类型的整数倍的意义：提升读取的效率。

```c
struct Man {
    int age;
    double weight;
};

void main() {
    struct Man m1 = { 20 ,89.9 };
    printf(" %#x , %d\n", &m1, sizeof(m1));
    getchar();
}

结果输出：
 0x95fd40 , 16
```

###### 本示例中 字节最长的基本数据类型为 double 8 位 * 2（成员数量） = 16

#### 结构体与动态内存分配

```c
void main() {
    struct Man *m_p = ( struct Man* )malloc(sizeof(struct Man) * 10);
    struct Man *p = m_p;

    //赋值
    p->name = "Jack";
    p->age  = 20;
    p++;
    p->name = "Rose";
    p->age = 20;

    struct Man *loop_p = m_p;
    for (; loop_p < m_p + 2;loop_p++) {
        printf("%s ,%d\n", loop_p->name, loop_p->age);
    }
    
    free(m_p);
    m_p = NULL;
    getchar();

}

结果输出：

Jack ,20
Rose ,20
```

#### typedef取别名，定义新的类型，方便使用

typedef 类型取别名

#### 用途一：

定义一种类型的别名，而不只是简单的宏替换。可以用作同时声明指针型的多个对象。比如：

```c
char* pa, pb; // 这多数不符合我们的意图，它只声明了一个指向字符变量的指针，和一个字符变量；
```

以下则可行：

```c
typedef char* PCHAR;
PCHAR pa, pb;
```

这种用法很有用，特别是char* pa, pb的定义，初学者往往认为是定义了两个字符型指针，其实不是，而用typedef char* PCHAR就不会出现这样的问题，减少了错误的发生。



#### 用途二：

用typedef来定义与平台无关的类型。  
比如定义一个叫 REAL 的浮点类型，在目标平台一上，让它表示最高精度的类型为：

```c
typedef long double REAL;
```

在不支持 long double 的平台二上，改为：

```c
typedef double REAL;
```

在连 double 都不支持的平台三上，改为：

```c
typedef float REAL;
```

也就是说，当跨平台时，只要改下 typedef 本身就行，不用对其他源码做任何修改。

标准库就广泛使用了这个技巧，比如size_t。另外，因为typedef是定义了一种类型的新别名，不是简单的字符串替换，所以它比宏来得稳健。  
这个优点在我们写代码的过程中可以减少不少代码量哦！

#### 用途三：

为复杂的声明定义一个新的简单的别名。方法是：在原来的声明里逐步用别名替换一部分复杂声明，如此循环，把带变量名的部分留到最后替换，得到的就是原声明的最简化版。

简单示例

```c
//Age int 类型的别名
typedef int age;
//Age int指针类型的别名
typedef int* Ap;


//写法一
struct Man
{
    char name[20];
    int age;
};

typedef struct Man JavaMan;
typedef struct Man* JM;

//----------------------------------------

//写法二
//结构体取别名
typedef struct Woman {
    char name[20];
    int age;

}W,*WP; //W 是Woman结构体的别名， WP 是Woman结构体指针的别名

//typedef struct Woman W; 
//typedef struct Woman* WP;

void main() {
    int i = 5;
    Ap p = &i;

    JavaMan m1 = { "Jack", 25 };
    JM jm1 = &m1;


    printf("%s, %d\n", m1.name, m1.age);

    printf("%s, %d\n", jm1->name, jm1->age);

    //结构体变量
    W w1 = { "Rose",20 };
    //结构体指针
    WP wp1 = &w1;

    printf("%s, %d\n", w1.name, w1.age);

    printf("%s, %d\n", wp1->name, wp1->age);

    getchar();
}

结果输出：

Jack, 25
Jack, 25
Rose, 20
Rose, 20
```

#### 结构体函数指针成员

```c
struct Girl {
    char *name;
    int age;
    //函数指针
    void(*sayHi)(char*);
};
 
void sayHi(char * text) {
    MessageBox(0,text, "title", 0);
}
//Girl 结构体类似于Java中类，name和age类似于属性，sayHi类似于方法;
void main() {
    struct Girl g1;
    g1.name = "Lucy";
    g1.age = 18;
    g1.sayHi = sayHi;
    g1.sayHi("HELLO");

    getchar();

}
结果输出：

弹出一个窗口
```

##### 别名结构体原本的名字相同时,struct Girl g1 可以简写为Girl g1 (类似于OOP思想)

```c
typedef struct Girl {
    char *name;
    int age;
    //函数指针
    void(*sayHi)(char*);
}Girl;//给结构体取一个别名Girl（别名可以与结构体原本的名字相同）

      //Girl结构体指针取别名GirlP
typedef Girl* GirlP;

//结构体的成员函数
void sayHi(char* text) {
    MessageBoxA(0, text, "title", 0);
}

//自定义的一个函数
void reName(GirlP gp1) {
    gp1->name = "Jack";
    gp1->age =  23;
}

void main() {
    Girl g1 = { "Lucy", 18, sayHi };
    printf("%s, %d\n", g1.name, g1.age);
    GirlP gp1 = &g1;

    //传递指针，改名（只有传递指针才能修改值，所以指针是比较常用的方式）
    reName(gp1);
    printf("%s, %d\n", g1.name, g1.age);
    gp1->sayHi("Byebye!");

    getchar();
}
结果输出：

Lucy, 18
Jack, 23
同时弹出一个窗口
```












