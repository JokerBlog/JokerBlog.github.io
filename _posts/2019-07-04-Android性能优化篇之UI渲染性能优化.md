---
layout:     post
title:      "Android性能优化篇之UI渲染性能优化"
subtitle:   ""
date:       2019-07-04 10:36:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---


> 在用户使用APP时，一方面想要华丽炫酷的动画交互，一方面需要交互的的流畅运行，如何平衡设计和性能就需要我们不断的学习和思考了。  
> UI渲染功能是最普通的功能，那么怎么衡量渲染性能的好坏？可能出现性能瓶颈的地方有哪些？造成卡顿的原因？如何解决卡顿？这些都是本章需要思考和解决的的问题。





### 1.关于ANR

#### 1.1 什么是ANR?

> ANR全名Application Not Responding, 也就是"应用无响应".当操作在一段时间内系统无法处理时, 系统层面会弹出ANR对话框.



#### 1.2 产生ANR的原因？

> APP的响应是Activity Manage和Window Manage来监控的，系统产生ANR的原因：
> 
> > - 5s内无法响应用户输入事件
> > > > - BoradCastReceiver在10s内没有处理结束  
> >   
> >   上面两点的根本原因就是主线程有耗时操作。



#### 1.3 如何避免？

> 1. 耗时操作放到子线程操作
> > 2. I/O操作放到子线程
> > 3. 避免内存泄漏（内存不够也会造成ANR，当时大多数情况是OOM）



#### 1.4 ANR如何分析？

> 导出/data/anr/下的traces.txt,发现日志来定位问题

```
adb pull data/anr/traces.txt ./
```



### 2.怎么衡量渲染性能的好坏？

###### 2.1 16ms

> 要知道Android系统每隔16ms就发出VSYNC信号重新绘制一次Activity,所以要在16ms内能够完成绘制，这样才能达到每秒60帧，然而这个每秒帧数的参数由手机硬件所决定，现在大多数手机屏幕刷新率是60赫兹（赫兹是国际单位制中频率的单位，它是每秒中的周期性变动重复次数的计量），也就是说我们有16ms（1000ms/60次=16.66ms）的时间去完成每帧的绘制逻辑操作，就不会出现卡顿的现象，如果没有完成，则会丢帧导致卡顿。



![](https://upload-images.jianshu.io/upload_images/5748654-ba148a7f47165ab5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/766/format/webp)



![](https://upload-images.jianshu.io/upload_images/5748654-e9b889057e4341a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/758/format/webp)





### 3.关于渲染管线

Android系统的渲染管线分为两个关键组件：CPU和GPU，它们共同工作，在屏幕上绘制图片，每个组件都有自身定义的特定流程。我们必须遵守这些特定的操作规则才能达到效果。

![](https://upload-images.jianshu.io/upload_images/5748654-0ff901067eb44354.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/500/format/webp)



CPU负责包括Measure，Layout，Record，Execute的计算操作，GPU负责Rasterization(栅格化)操作。  
在CPU方面，最常见的性能问题是不必要的布局和失效，这些内容必须在视图层次结构中进行测量、清除并重新创建，引发这种问题通常有两个原因：一是重建显示列表的次数太多，二是花费太多时间作废视图层次并进行不必要的重绘，这两个原因在更新显示列表或者其他缓存GPU资源时导致CPU工作过度。  

在GPU方面，最常见的问题是我们所说的过度绘制（overdraw），通常是在像素着色过程中，通过其他工具进行后期着色时浪费了GPU处理时间。



###### 3.1 GPU

> 了解Android中如何使用GPU进行画面的渲染可以帮助我们更好的理解性能问题。我们的布局文件是如何被绘制到屏幕上的？



![](https://upload-images.jianshu.io/upload_images/5748654-ddf0b2ff9595be4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/395/format/webp)



Resterization栅格化是绘制那些Button，Shape，Path，String，Bitmap等组件最基础的操作。它把那些组件拆分到不同的像素上进行显示。这是一个很费时的操作，GPU的引入就是为了加快栅格化的操作。



![](https://upload-images.jianshu.io/upload_images/5748654-1cb85152139154c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/378/format/webp)



GPU使用一些指定的基础指令集，主要是多边形和纹理，也就是图片，CPU在屏幕上绘制图像前会向GPU输入这些指令，这一过程通常使用的API就是Android的OpenGL ES，这就是说，在屏幕上绘制UI对象时无论是按钮、路径或者复选框，都需要在CPU中首先转换为多边形或者纹理，然后再传递给GPU进行格栅化。  
UI对象转换为一系列多边形和纹理的过程肯定相当耗时，从CPU上传处理数据到GPU同样也很耗时。所以很明显，我们需要尽量减少对象转换的次数，以及上传数据的次数，幸亏，OpenGL ES API允许数据上传到GPU后可以对数据进行保存，当我们下次绘制一个按钮时，只需要在GPU存储器里引用它，然后告诉OpenGL如何绘制就可以了，一条经验之谈：

###### 渲染性能的优化就是尽可能地上传数据到GPU，然后尽可能长地在不修改的情况下保存数据，因为每次上传资源到GPU时，我们都会浪费宝贵的处理时间.

为了能够使得App流畅，我们需要在每帧16ms以内处理完所有的CPU与GPU的计算，绘制，渲染等等操作。



### 4.Hierarchy Viewer工具介绍

![](https://upload-images.jianshu.io/upload_images/5748654-4e478008066f2688.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/903/format/webp)



(2).选择Hierarchy Viewer选项卡

![](https://upload-images.jianshu.io/upload_images/5748654-fd5fcc0a8952b5ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



#### 4.1 设备连接问题

如果你是用的模拟器或者开发版手机的话则可以直接进行连接调试了，如果不是的话，官方提供了两种方式，进行连接真机调试：

第一种，通过第三方库，安装和配置ViewServer,也是目前我在使用的方式，工具地址：[点击](https://github.com/romainguy/ViewServer)，步骤如下：  
(1).添加依赖  
project 下的 build.gradle

```
    allprojects {  
            repositories {  
                jcenter()  
                maven { url "https://jitpack.io" }  
            }  
        }
```

module 下的 build.gradle

```
    dependencies {  
            ...  
            compile 'com.github.romainguy:ViewServer:017c01cd512cac3ec054d9eee05fc48c5a9d2de'  
        }
        
```

(2).申请权限

```
   <uses-permission android:name="android.permission.INTERNET"/>
  
```

(3).添加代码

```
      public void onCreate(Bundle savedInstanceState) {  
            super.onCreate(savedInstanceState);  
            ViewServer.get(this).addWindow(this);  
        }  
        public void onDestroy() {  
            super.onDestroy();  
            ViewServer.get(this).removeWindow(this);  
        }  

        public void onResume() {  
            super.onResume();  
            ViewServer.get(this).setFocusedWindow(this);  
        }
```

第二种，通过设置环境变量，export ANDROID_HVPROTO=ddm（可能对小米手机无用）



#### 4.2 性能表示

这里我们主要关注下面的三个圆圈，从左到右依次，代表View的Measure, Layout和Draw的性能，不同颜色代表不同的性能等级：



(1). 绿: 表示该View的此项性能比该View Tree中超过50%的View都要快；例如,一个绿点的测量时间意味着这个视图的测量时间快于树中的视图对象的50%。



(2). 黄: 表示该View的此项性能比该View Tree中超过50%的View都要慢；例如,一个黄点布局意味着这种观点有较慢的布局时间超过50%的树视图对象。



(3). 红: 表示该View的此项性能是View Tree中最慢的；例如,一个红点的绘制时间意味着花费时间最多的这一观点在树上画所有的视图对象。



### 5.问题分析以及解决方案

#### 5.1 CPU

上面已经分析过了，CPU常见的性能问题是不必要的布局和失效，引发这种问题通常有两个原因：一是重建显示列表的次数太多，二是花费太多时间作废视图层次并进行不必要的重绘。

#### 5.1.1 布局失效优化

Android需要把XML布局文件转换成GPU能够识别并绘制的对象。这个操作是在DisplayList的帮助下完成的。DisplayList持有所有将要交给GPU绘制到屏幕上的数据信息,还有执行绘制操作的OpenGL命令列表。在某个View第一次需要被渲染时，Display List会因此被创建，当这个View要显示到屏幕上时，我们会执行GPU的绘制指令来进行渲染。  
那么第二次渲染这个view会发生什么呢？

##### 1.如果View的Property属性发生了改变（例如移动位置），我们就仅仅需要Execute Display List就够了.

![](https://upload-images.jianshu.io/upload_images/5748654-a30db961bf30fa94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/313/format/webp)



##### 2.如果你修改了View中的某些可见组件的内容，那么之前的DisplayList就无法继续使用了，我们需要重新创建一个DisplayList并重新执行渲染指令更新到屏幕上。任何时候View中的绘制内容发生变化时，都会需要重新创建DisplayList，渲染DisplayList，更新到屏幕上等一系列操作。这个流程的表现性能取决于你的View的复杂程度，View的状态变化以及渲染管道的执行性能。

![](https://upload-images.jianshu.io/upload_images/5748654-790c1e972b0ab8e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/312/format/webp)



#### 3.如果某个View的大小需要增大到目前的两倍，在增大View大小之前，需要通过父View重新计算并摆放其他子View的位置。修改View的大小会触发整个HierarcyView的重新计算大小的操作。

![](https://upload-images.jianshu.io/upload_images/5748654-c2845ee6fce36d72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/437/format/webp)



#### 5.1.2 嵌套结构优化

提升布局性能的关键点是尽量保持布局层级的扁平化，避免出现重复的嵌套布局。  
我们先来看下列子，然后再来总结：  
当前页面有两个条目，上面条目是使用LinearLayout中嵌套LinearLayout实现的，下面条目使用一个RelativeLayout实现

![](https://upload-images.jianshu.io/upload_images/5748654-ea9023c8f7ce89fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/375/format/webp)





我们使用Hierarchy Viewer工具来看下：

![](https://upload-images.jianshu.io/upload_images/5748654-b2692a3d5357fba9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/746/format/webp)



现在我们把上面一个条目改成和下面条目一样的实现，看下优化后的效果：

![](https://upload-images.jianshu.io/upload_images/5748654-03a1bdba742d6e44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/536/format/webp)



#### 5.1.2.1 结果分析

红色节点是代表应用性能慢的一个潜在问题，下面是几个例子，如何来分析和解释红点的出现原因？

##### (1).如果在叶节点或者ViewGroup中，只有极少的子节点，这可能反映出一个问题，应用可能在设备上运行并不慢，但是你需要指导为什么这个节点是红色的，可以借助Systrace或者Traceview工具，获取更多额外的信息

##### (2).如果一个视图组里面有许多的子节点，并且测量阶段呈现为红色，则需要观察下子节点的绘制情况

##### (3).如果视图层级结构中的根视图，Messure阶段为红色，Layout阶段为红色，Draw阶段为黄色，这个是比较常见的，因为这个节点是所有其它视图的父类

##### (4).如果视图结构中的一个叶子节点，有20个视图是红色的Draw阶段，这是有问题的，需要检查代码里面的onDraw方法，不应该在那里调用



#### 5.1.2.2 优化建议

#### (1).没有用的父布局时指没有背景绘制或者没有大小限制的父布局，这样的布局不会对UI效果产生任何影响。我们可以把没有用的父布局，通过<merge/>标签合并来减少UI的层次

##### (2).使用线性布局LinearLayout排版导致UI层次变深，如果有这类问题，我们就使用相对布局RelativeLayout代替LinearLayout,减少UI的层次

##### (3).不常用的UI被设置成GONE,比如异常的错误页面，如果有这类问题，我们需要用<ViewStub/>标签，代替GONE提高UI性能



#### 5.1.3 常用的优化示例

##### (1). include 标签

include标签常用于将布局中的公共部分提取出来供其他layout共用，以实现布局模块化，这在布局编写方便提供了大大的便利。

##### (2). viewstub 标签

viewstub标签同include标签一样可以用来引入一个外部布局，不同的是，viewstub引入的布局默认不会扩张，即既不会占用显示也不会占用位置，从而在解析layout时节省cpu和内存。  
viewstub常用来引入那些默认不会显示，只在特殊情况下显示的布局，如进度布局、网络失败显示的刷新布局、信息出错出现的提示布局等。



```java
    //第一种
    ViewStub stub = (ViewStub)findViewById(...)
    View stubView=  stub.inflate();
    //根据实际情况，显示
    stubView.setVisibility()

    //第二种
    View viewStub = findViewById(R.id.network_error_layout);
    viewStub.setVisibility(View.VISIBLE);   // ViewStub被展开后的布局所替换
```



##### 3). merge 标签

在使用了include后可能导致布局嵌套过多，多余不必要的layout节点，从而导致解析变慢

##### merge标签可用于两种典型情况：

(1). 布局顶结点是FrameLayout且不需要设置background或padding等属性，可以用merge代替，因为Activity内容试图的parent view就是个FrameLayout，所以可以用merge消除只剩一个。  
(2). 某布局作为子布局被其他布局include时，使用merge当作该布局的顶节点，这样在被引入时顶结点会自动被忽略，而将其子节点全部合并到主布局中。



#### 5.2 GPU

在GPU方面，最常见的问题是我们所说的过度绘制（overdraw），通常是在像素着色过程中，通过其他工具进行后期着色时浪费了GPU处理时间。

![](https://upload-images.jianshu.io/upload_images/5748654-bef546564b4e63ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/450/format/webp)



过度绘制描述的是屏幕上的某个像素在同一帧的时间内被绘制了多次。在多层次重叠的UI结构里面，如果不可见的UI也在做绘制的操作，会导致某些像素区域被绘制了多次。这样就会浪费大量的CPU以及GPU资源。  
当设计上追求更华丽的视觉效果的时候，我们就容易陷入采用复杂的多层次重叠视图来实现这种视觉效果的怪圈。这很容易导致大量的性能问题，为了获得最佳的性能，我们必须尽量减少Overdraw的情况发生。

幸运的是，我们可以通过手机设置里面的开发者选项，打开Show GPU Overdraw的选项，观察UI上的Overdraw情况。

![](https://upload-images.jianshu.io/upload_images/5748654-9bde4b7bbb411466.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/382/format/webp)

##### GPU Profiling

从Android M系统开始，系统更新了GPU Profiling的工具来帮助我们定位UI的渲染性能问题。早期的CPU Profiling工具只能粗略的显示出Process，Execute，Update三大步骤的时间耗费情况。

![](https://upload-images.jianshu.io/upload_images/5748654-810ded64abf135a2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/506/format/webp)

但是仅仅显示三大步骤的时间耗费情况，还是不太能够清晰帮助我们定位具体的程序代码问题，所以在Android M版本开始，GPU Profiling工具把渲染操作拆解成如下8个详细的步骤进行显示。



![](https://upload-images.jianshu.io/upload_images/5748654-a1c306f4b95f2158.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/407/format/webp)





旧版本中提到的Proces，Execute，Update还是继续得到了保留，他们的对应关系如下：

![](https://upload-images.jianshu.io/upload_images/5748654-140272d40183017b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/495/format/webp)



- Sync & Upload：通常表示的是准备当前界面上有待绘制的图片所耗费的时间，为了减少该段区域的执行时间，我们可以减少屏幕上的图片数量或者是缩小图片本身的大小。
- Measure & Layout：这里表示的是布局的onMeasure与onLayout所花费的时间，一旦时间过长，就需要仔细检查自己的布局是不是存在严重的性能问题。
- Animation：表示的是计算执行动画所需要花费的时间，包含的动画有ObjectAnimator，ViewPropertyAnimator，Transition等等。一旦这里的执行时间过长，就需要检查是不是使用了非官方的动画工具或者是检查动画执行的过程中是不是触发了读写操作等等。
- Input Handling：表示的是系统处理输入事件所耗费的时间，粗略等于对于的事件处理方法所执行的时间。一旦执行时间过长，意味着在处理用户的输入事件的地方执行了复杂的操作。
- Misc/Vsync Delay：如果稍加注意，我们可以在开发应用的Log日志里面看到这样一行提示：I/Choreographer(691): Skipped XXX frames! The application may be doing too much work on its main thread。这意味着我们在主线程执行了太多的任务，导致UI渲染跟不上vSync的信号而出现掉帧的情况。

上面八种不同的颜色区分了不同的操作所耗费的时间，为了便于我们迅速找出那些有问题的步骤，GPU Profiling工具会显示16ms的阈值线，这样就很容易找出那些不合理的性能问题，再仔细看对应具体哪个步骤相对来说耗费时间比例更大，结合上面介绍的细化步骤，从而快速定位问题，修复问题。



#### 5.2.1 优化建议

##### (1).移除Window默认的Background

```java
    getWindow().setBackgroundDrawable(null);
```

##### (2).移除XML布局文件中非必需的Background

##### (3).按需显示占位背景图片

在给ImageView设置图片时，判断是否获取到对应的Bitmap，在获取到图像之后，把ImageView的Background设置为Transparent，只有当图像没有获取到的时候才设置对应的Background占位图片，这样可以避免因为设置背景图而导致的过度渲染。

##### (4).剪辑不显示的UI组件

对不可见的UI组件进行绘制更新会导致Overdraw。例如Nav Drawer从前置可见的Activity滑出之后，如果还继续绘制那些在Nav Drawer里面不可见的UI组件，这就导致了Overdraw。为了解决这个问题，Android系统会通过避免绘制那些完全不可见的组件来尽量减少Overdraw。那些Nav Drawer里面不可见的View就不会被执行浪费资源。



![](https://upload-images.jianshu.io/upload_images/5748654-b7c6a120baeb140c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/467/format/webp)





但是不幸的是，对于那些过于复杂的自定义的View(通常重写了onDraw方法)，Android系统无法检测在onDraw里面具体会执行什么操作，系统无法监控并自动优化，也就无法避免Overdraw了。但是我们可以通过canvas.clipRect()来帮助系统识别那些可见的区域。这个方法可以指定一块矩形区域，只有在这个区域内才会被绘制，其他的区域会被忽视。这个API可以很好的帮助那些有多组重叠组件的自定义View来控制显示的区域。同时clipRect方法还可以帮助节约CPU与GPU资源，在clipRect区域之外的绘制指令都不会被执行，那些部分内容在矩形区域内的组件，仍然会得到绘制。  
除了clipRect方法之外，我们还可以使用canvas.quickreject()来判断是否没和某个矩形相交，从而跳过那些非矩形区域内的绘制操作。  
下面我们来看个实例：

![](https://upload-images.jianshu.io/upload_images/5748654-1645eefe2a0912a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/407/format/webp)





代码：

```java
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (mCardList != null && mCardList.size() > 0) {
            for (int i = 0; i < mCardList.size(); i++) {
                mCardLeft = i * mCardSpacing;
                drawCard(canvas, mCardList.get(i), mCardLeft, 0);
            }
        }
    }

    private void drawCard(Canvas canvas, CardItem card, int left, int top) {
        Bitmap mBitmap = getBitmap(card.resId);
        canvas.drawBitmap(mBitmap, left, top, mPaint);
    }

    private Bitmap getBitmap(int resId) {
        return BitmapFactory.decodeResource(this.getResources(), resId);
    }
```

我们看到扑克牌有不可见的区域但是还是被绘制了，导致过度绘制。下面我们进行剪辑。

![](https://upload-images.jianshu.io/upload_images/5748654-6b9f44aab21843f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/398/format/webp)









代码：

```java
@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (mCardList != null && mCardList.size() > 0) {
            for (int i = 0; i < mCardList.size()-1; i++) {
                mCardLeft = i * mCardSpacing;
                canvas.save();
                canvas.clipRect(mCardLeft,
                        0f,
                        mCardLeft + mCardSpacing,
                        mCardList.get(i).getHeight());
                drawCard(canvas, mCardList.get(i), mCardLeft, 0);
                canvas.restore();
            }
            drawCard(canvas, mCardList.get(mCardList.size()-1), mCardLeft + mCardSpacing, 0);
        }
    }
```

###### 5.3 其他问题引起的卡顿分析

###### 5.3.1 内存抖动

内存抖动是因为在短时间内大量的对象被创建又马上被释放。瞬间产生大量的对象会严重占用Young Generation的内存区域，当达到阀值，剩余空间不够的时候，会触发GC从而导致刚产生的对象又很快被回收。即使每次分配的对象占用了很少的内存，但是他们叠加在一起会增加Heap的压力，从而触发更多其他类型的GC。这个操作有可能会影响到帧率，并使得用户感知到性能问题(卡顿)。

![](https://upload-images.jianshu.io/upload_images/5748654-8dd68fbb5456431a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/532/format/webp)





解决上面的问题有简洁直观方法，如果你在Memory Monitor里面查看到短时间发生了多次内存的涨跌，这意味着很有可能发生了内存抖动

![](https://upload-images.jianshu.io/upload_images/5748654-0134a6fdb7ebcd43.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/464/format/webp)



同时我们还可以通过*Allocation Tracker*来查看在短时间内，同一个栈中不断进出的相同对象。这是内存抖动的典型信号之一。

当你大致定位问题之后，接下去的问题修复也就显得相对直接简单了。例如，你需要避免在for循环里面分配对象占用内存，需要尝试把对象的创建移到循环体之外，自定义View中的onDraw方法也需要引起注意，每次屏幕发生绘制以及动画执行过程中，onDraw方法都会被调用到，避免在onDraw方法里面执行复杂的操作，避免创建对象。对于那些无法避免需要创建对象的情况，我们可以考虑对象池模型，通过对象池来解决频繁创建与销毁的问题，但是这里需要注意结束使用之后，需要手动释放对象池中的对象。












