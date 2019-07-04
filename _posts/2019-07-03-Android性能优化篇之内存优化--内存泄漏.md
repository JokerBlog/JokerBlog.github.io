---
layout:     post
title:      "Android性能优化篇之内存优化--内存泄漏"
subtitle:   ""
date:       2019-07-02 15:46:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---


今天主要是讲内存泄漏的产生原因分析，常见的导致内存泄漏的示例，以及内存泄漏优化的方法。中间穿插着有关java虚拟机内存管理，内存分配策略，垃圾收集器的相关知识点。下面就来列出今天讲解的大体流程。



### 讲解流程：

1.什么是内存泄漏？  
2.android中导致内存泄漏的主要几个点  
3.java虚拟机内存管理  
4.java内存几种分配策略？  
5.垃圾收集器是如何判断对象是否可回收？  
6.什么是内存抖动？  
7.内存抖动产生的原因？  
8.android中4种引用  
9.常见的导致内存泄漏的示例



### 1.什么是内存泄漏？

当一个对象已经不需要在使用了，本应该被回收，而另一个正在使用的对象持有它的引用，导致对象不能被回收。因为不能被及时回收的本该被回收的内存，就产生了内存泄漏。如果内存泄漏太多会导致程序没有办法申请内存，最后出现内存溢出的错误。



### 2.android中导致内存泄漏的主要几个点

android开发中经常出现的点，我有只有了解了，才能更好的避免。

> - 使用单例模式
> > - 使用匿名内部类
> > - 使用异步事件处理机制Handler
> > - 使用静态变量
> > - 资源未关闭
> > - 设置监听
> > - 使用AsyncTask
> > - 使用Bitmap



#### 3.java虚拟机内存管理

![](https://upload-images.jianshu.io/upload_images/5748654-9cb41898d70dda16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/705/format/webp)

java虚拟机内存分为虚拟机栈，本地方法栈，程序计数器，堆，方法区这几个模块，下面我们就来分析下各个模块。

##### (1).虚拟机栈

- 虚拟机栈主要的作用就是为执行java方法服务的，是Java方法执行的动态内存模型。
- 线程私有，生命周期和线程一致。描述的是 Java 方法执行的内存模型：每个方法在执行时都会床创建一个栈帧(Stack Frame)用于存储`局部变量表`、`操作数栈`、`动态链接`、`方法出口`等信息。每一个方法从调用直至执行结束，就对应着一个栈帧从虚拟机栈中入栈到出栈的过程。
- 局部变量表：存放了编译期可知的各种基本类型(boolean、byte、char、short、int、float、long、double)、对象引用(reference 类型)和 returnAddress 类型(指向了一条字节码指令的地址)

  ##### 会导致栈内存溢出(StackOverFlowError)

##### (2).本地方法栈

- 为执行native方法服务的，其他和虚拟机栈一样
- 区别于 Java 虚拟机栈的是，Java 虚拟机栈为虚拟机执行 Java 方法(也就是字节码)服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。也会有 StackOverflowError 和 OutOfMemoryError 异常。
- 

##### (3).程序计数器

- 是当前线程执行的字节码行号指示器
- 处于线程独占区
- 如果是执行的是java代码,当前值为字节码指令的地址，如果是Native，值为undefined
- 内存空间小，线程私有。字节码解释器工作是就是通过改变这个计数器的值来选取下一条需要执行指令的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖计数器完成

##### (4).堆

- 存放对象的实例
- 垃圾收集器管理的主要区域
- 分代管理对象
- 会导致内存溢出(OutOfMemoryError)
- 对于绝大多数应用来说，这块区域是 JVM 所管理的内存中最大的一块。线程共享，主要是存放对象实例和数组。内部会划分出多个线程私有的分配缓冲区(Thread Local Allocation Buffer, TLAB)。可以位于物理上不连续的空间，但是逻辑上要连续。



##### (5).本地方法栈

- 为执行native方法服务的，其他和虚拟机栈一样
- 属于共享内存区域，存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。



### 4.java内存几种分配策略？

可以结合上面的内存分配模型，能很好的理解。

##### (1).静态的

- 静态存储区：内存在程序编译期间就已经分配完成，一般来说，这个区域在程序运行期间一直处在
- 它主要储存静态数据，全局静态数据和常量



##### (2).栈式的

- 执行方法时，存储局部变量(编译期间，已经确定占用内存大小)，操作数，动态链接，方法出口



##### (3).堆式的

- 也叫动态内存分配，主要存储对象实例，以及已经被加载类的Class对象(用于反射)



### 5.垃圾收集器是如何判断对象是否可回收？

我们知道内存泄漏的原因是应该被回收的对象，不能被及时回收，那么GC是如何来判断对象是否为垃圾对象呢？

判断的方式有两个：

###### * 引用计数

> 对象被引用，引用计数器加1，反之减一，只有引用计数为0,那么这个对象为垃圾对象



###### * 可达性

> 从GCRoot节点对象开始，看是否可以访问到此对象，如果没有访问到则为垃圾对象



#### 可以作为GCRoot对象有以下几种：

> - 虚拟机栈中的局部变量
> > - 本地方法栈中的引用对象
> > - 方法区中的常量引用对象
> > - 方法区中的类属性引用对象



在native层和早期的虚拟机一般使用引用计数，但是现在的java虚拟机大多使用的是可达性。



### 6.什么是内存抖动？

> 堆内存都有一定的大小，能容纳的数据是有限制的，当Java堆的大小太大时，垃圾收集会启动停止堆中不再应用的对象，来释放内存。当在极短时间内分配给对象和回收对象的过程就是内存抖动。



### 7.内存抖动产生的原因？

> 从术语上来讲就是极短时间内分配给对象和回收对象的过程。  
> 一般多是在循环语句中创建临时对象，在绘制时配置大量对象或者执行动画时创建大量临时对象  
> 内存抖动会带来UI的卡顿，因为大量的对象创建，会很快消耗剩余内存，导致GC回收，GC会占用大量的帧绘制时间，从而导致UI卡顿，关于UI卡顿会在后面章节讲到。




### 8.android中四种引用

> ###### (1).StrongReference强引用
> 
> > 从不被回收，java虚拟机停止时，才终止
> 
> ###### (2).SoftReference软引用
> 
> > 当内存不足时，会主动回收，使用SoftReference使用结合ReferenceQueue构造有效期短
> 
> ###### (3).WeakReference弱引用
> 
> > 每次垃圾回收时，被回收
> 
> ###### (4).PhatomReference虚引用
> 
> > 每次垃圾回收时，被回收.结合ReferenceQueue来跟踪对象被垃圾回收器回收的活动





### 9.常见的导致内存泄漏的示例

#### (1).使用单例模式

```java
  private static ComonUtil mInstance = null;
    private Context mContext = null;

    public ComonUtil(Context context) {
        mContext = context;
    }

    public static ComonUtil getInstance(Context context) {
        if (mInstance == null) {
            mInstance = new ComonUtil(context);
        }
        return mInstance;
    }
```

使用：

```java
ComonUtil mComonUtil = ComonUtil.getInstance(this);
```

我们看到上面的代码就是我们平时使用的单例模式，当然这里没有考虑线程安全，请忽略。当我们传递进来的是Context，那么当前对象就会持有第一次实例化的Context，如果Context是Activity对象，那么就会产生内存泄漏。因为当前对象ComonUtil是静态的，生命周期和应用是一样的，只有应用退出才会释放，导致Activity不能及时释放，带来内存泄漏。

###### 怎么解决呢？

常见的有两种方式，第一就是传入ApplicationContext，第二CommonUtil中取context.getApplicationContext()。

```java
   public ComonUtil(Context context) {
        mContext = context.getApplicationContext();
    }
```

#### (2).使用非静态内部类

```java
   /**
     * 非静态内部类
     */
    public void createNonStaticInnerClass(){
        CustomThread mCustomThread = new CustomThread();
        mCustomThread.start();
    }

    public class CustomThread extends Thread{
        @Override
        public void run() {
            super.run();
            while (true){
                try {
                    Thread.sleep(5000);
                    Log.i(TAG,"CustomThread ------- 打印");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```

我们就以线程为例，当Activity调用了createNonStaticInnerClass方法，然后退出当前Activity时，因为线程还在后台执行且当前线程持有Activity引用，只有等到线程执行完毕，Activitiy才能得到释放，导致内存泄漏。  
常用的解决方法有很多，第一把线程类声明为静态的类，如果要用到Activity对象，那么就作为参数传入且为WeakReference,第二在Activity的onDestroy时，停止线程的执行。

```java
    public static class CustomThread extends Thread{
        private WeakReference<MainActivity> mActivity;
        public CustomThread(MainActivity activity){
            mActivity = new WeakReference<MainActivity>(activity)
        }
    }
```

#### (3).使用异步事件处理机制Handler

```java
 /**
     * 异步消息处理机制  -- handler机制
     */
    public void createHandler(){
        mHandler.sendEmptyMessage(0);
    }
    public Handler mHandler = new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            //处理耗时操作   
            return false;
        }
    });
```

这个应该是我们平时使用最多的一种方式，如果当handler中处理的是耗时操作，或者当前消息队列中消息很多时，那当Activity退出时，当前message中持有handler的引用，handler又持有Activity的引用，导致Activity不能及时的释放，引起内存泄漏的问题。  

解决handler引起的内存泄漏问题常用的两种方式：  
1.和上面解决Thread的方式一样，  
2.在onDestroy中调用mHandler.removeCallbacksAndMessages(null)

```java
 @Override
    protected void onDestroy() {
        super.onDestroy();
        mHandler.removeCallbacksAndMessages(null);
    }
```



#### (4).使用静态变量

> 同单例引起的内存泄漏。



#### (5).资源未关闭

> 常见的就是数据库游标没有关闭，对象文件流没有关闭，主要记得关闭就OK了。



#### (6).设置监听

> 常见的是在观察者模式中出现，我们在退出Acviity时没有取消监听，导致被观察者还持有当前Activity的引用，从而引起内存泄漏。  
> 常见的解决方法就是在onPause中注消监听



#### (7).使用AsyncTask

```java
public AsyncTask<Object, Object, Object> mTask = new AsyncTask<Object, Object, Object>() {

        @Override
        protected Object doInBackground(Object... params) {
            //耗时操作
            return null;
        }

        @Override
        protected void onPostExecute(Object o) {
        
        }   
    };
```

和上面同样的道理，匿名内部类持有外部类的引用，AsyncTask耗时操作导致Activity不能及时释放，引起内存泄漏。  
解决方法同上:  
1.声明为静态类，  
2.在onPause中取消任务



#### (8).使用Bitmap

> 我们知道当bitmap对象没有被使用(引用)，gc会回收bitmap的占用内存，当时这边的内存指的是java层的，那么本地内存的释放呢？我们可以通过调用bitmap.recycle()来释放C层上的内存，防止本地内存泄漏














