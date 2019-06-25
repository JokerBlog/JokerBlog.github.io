---
layout:     post
title:      "从Activity启动流程聊Binder机制"
subtitle:   ""
date:       2019-06-25 10:07:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### 1、  Activity启动流程

Activity启动流程分两种，一种是启动正在运行的app的Activity，即启动子Activity。如无特殊声明默认和启动该activity的activity处于同一进程。如果有声明在一个新的进程中，则处于两个进程。另一种是打开新的app，即为Launcher启动新的Activity。后边启动Activity的流程是一样的，区别是前边判断进程是否存在的那部分。  

Activity启动的前提是已经开机，各项进程和AMS等服务已经初始化完成，在这里也提一下那些内容。



### 2、 Activity启动之前

**init进程**：init是所有linux程序的起点，是Zygote的父进程。解析init.rc孵化出Zygote进程。

**Zygote进程**：Zygote是所有Java进程的父进程，所有的App进程都是由Zygote进程fork生成的。

**SystemServer进程**：System Server是Zygote孵化的第一个进程。SystemServer负责启动和管理整个Java framework，包含AMS，PMS等服务。

**Launcher**：Zygote进程孵化的第一个App进程是Launcher



#### 顺序是Linux--->init--->Zygote--->SystemServer---->Launcher



### 3、 init进程是什么？

Android是基于linux系统的，手机开机之后，linux内核进行加载。加载完成之后会启动init进程。  
init进程会启动ServiceManager，孵化一些守护进程，并解析init.rc孵化Zygote进程。



### 4、Zygote进程是什么？

所有的App进程都是由Zygote进程fork生成的，包括SystemServer进程。Zygote初始化后，会注册一个等待接受消息的socket，OS层会采用socket进行IPC通信。



### 5、为什么是Zygote来孵化进程，而不是新建进程呢？

每个应用程序都是运行在各自的Dalvik虚拟机中，应用程序每次运行都要重新初始化和启动虚拟机，这个过程会耗费很长时间。Zygote会把已经运行的虚拟机的代码和内存信息共享，起到一个预加载资源和类的作用，从而缩短启动时间



### 6、Activity启动阶段

**进程**：Android系统为每个APP分配至少一个进程

**IPC**：跨进程通信，Android中采用Binder机制。[跳转到从AIDL聊Binder]([https://jokerblog.github.io/2019/06/18/Android-Binder解密/](https://jokerblog.github.io/2019/06/18/Android-Binder%E8%A7%A3%E5%AF%86/))



直观来说，Binder是Android中的一个类，它实现了IBinder接口。从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有。从Android Framework角度来说，Binder是ServiceManager连接各种Manager(ActivityManager、WindowManager等等)和相应ManagerService的桥梁。从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。



#### Binder的组成

Binder由四部分组成：Binder客户端、Binder服务端、Binder驱动、服务登记查询模块。

- Binder客户端：

Binder客户端是想要使用服务的进程。

- Binder服务端：

Binder服务端是实际提供服务的进程。

- Binder驱动：

我们在客户端先通过Binder拿到一个服务端进程中的一个对象的引用，通过这个引用，直接调用对象的方法获取结果。在这个引用对象执行方法时，它是先将方法调用的请求传给Binder驱动；然后Binder驱动再将请求传给服务端进程；服务端进程收到请求后，调用服务端“真正”的对象来执行所调用的方法；得出结果后，将结果发给Binder驱动；Binder驱动再将结果发给我们的客户端；最终，我们在客户端进程的调用就有了返回值。Binder驱动，相当于一个中转者的角色。通过这个中转者的帮忙，我们就可以调用其它进程中的对象。

- Service Manager（服务登记查询模块）：

我们调用其它进程里面的对象时，首先要获取这个对象。这个对象其实代表了另外一个进程能给我们提供什么样的服务（再直接一点，就是：对象中有哪些方法可以让客户端进程调用）。首先服务端进程要在某个地方注册登记一下，告诉系统我有个对象可以公开给其它进程来提供服务。当客户端进程需要这个服务时，就去这个登记的地方通过查询来找到这个对象。



#### Binder工作流程

　    假设：客户端的程序Client运行在进程A中，服务端的程序Server运行在进程B中。

　　由于进程的隔离性，Client不能读写Server中的内容，但内核可以，而Binder驱动就是运行在内核态，因此Binder驱动帮我们进行请求的中转。

　　有了Binder驱动，Client和Server之间就可以打交道了，但是为了实现功能的单一性，我们为Client和Server分别设置一个代理：Client的代理Proxy和Server的代理Stub。这样，由进程A中的Proxy和进程B中的Stub通过Binder驱动进行数据交流，Server和Client直接调用Stub和Proxy的接口返回数据即可。

　　此时，Client直接调用Proxy这个聚合了Binder的类，我们可以使用一系列的Manager来屏蔽掉Binder的实现细节，Client直接调用Manager中的方法获取数据，这样做的好处是Client不需要知道Binder具体怎么工作。

　　最后还有一个问题，就是Client想要获得的服务多种多样，那么它是怎么获取Proxy或Manager的呢？答案是通过Service Manager进程来获取的。Service Manager总是第一个启动的服务，其他服务端进程启动后，可以在Service Manager中注册，这样Client就可以通过Service Manager来获取服务器的服务列表，进而选择具体调用的服务器进程方法。

![](https://images2015.cnblogs.com/blog/1083990/201703/1083990-20170329153813748-608849987.png)



#### 题外话：Android的其他Binder方式对比

![](https://images2015.cnblogs.com/blog/331079/201701/331079-20170122173430285-1744831757.jpg)





补充一下Messager的原理

![](https://images2015.cnblogs.com/blog/331079/201701/331079-20170122173414410-1107539422.jpg)

Messenger可以翻译为信使，顾名思义，通过它可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递。Messenger是一种轻量级的IPC方案，它的底层实现是AIDL，实现Messenger有以下两个步骤，分为服务端进程和客户端进程。



#### 回到正题 启动流程涉及到的类

**ActivityStack**：Activity在AMS的栈管理，用来记录已经启动的Activity的先后关系，状态信息等。通过ActivityStack决定是否需要启动新的进程。

**ActivitySupervisor**：管理 activity 任务栈

**ActivityThread**：ActivityThread 运行在UI线程（主线程），App的真正入口。

**ApplicationThread**：用来实现AMS和ActivityThread之间的交互。

**ApplicationThreadProxy**：ApplicationThread 在服务端的代理。AMS就是通过该代理与ActivityThread进行通信的。

**IActivityManager**：继承与IInterface接口，抽象出跨进程通信需要实现的功能

**AMN**：运行在server端（SystemServer进程）。实现了Binder类，具体功能由子类AMS实现。

**AMS**：AMN的子类，负责管理四大组件和进程，包括生命周期和状态切换。AMS因为要和ui交互，所以极其复杂，涉及window。

**AMP**：AMS的client端代理（app进程）。了解Binder知识可以比较容易理解server端的stub和client端的proxy。AMP和AMS通过Binder通信。

**Instrumentation**：仪表盘，负责调用Activity和Application生命周期。测试用到这个类比较多。



#### 流程图

![](https://mmbiz.qpic.cn/mmbiz_png/aYAPxAQvdVa3KqbNdFMGxpUqhYWdicuGXdSrrSHmz6cnlrKWad1sHmJeBa36ptNFglia7I4Q2qZ9ibXOJPpavgKKw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 涉及到的进程

- Launcher所在的进程

- AMS所在的SystemServer进程

- 要启动的Activity所在的app进程

如果是启动根Activity，就涉及上述三个进程。  
如果是启动子Activity，那么就只涉及AMS进程和app所在进程。



#### 具体流程

1. Launcher：Launcher通知AMS要启动activity。

- startActivitySafely->startActivity->Instrumentation.execStartActivity()(AMP.startActivity)->AMS.startActivity

2. AMS:PMS的resoveIntent验证要启动activity是否匹配。如果匹配，通过ApplicationThread发消息给Launcher所在的主线程，暂停当前Activity(即Launcher)。

3. 暂停完，在该activity还不可见时，通知AMS，根据要启动的Activity配置ActivityStack。然后判断要启动的Activity进程是否存在?

- 存在：发送消息LAUNCH_ACTIVITY给需要启动的Activity主线程，执行handleLaunchActivity

- 不存在：通过socket向zygote请求创建进程。进程启动后，ActivityThread.attach

4. 判断Application是否存在，若不存在，通过LoadApk.makeApplication创建一个。在主线程中通过thread.attach方法来关联ApplicationThread。

5. 在通过ActivityStackSupervisor来获取当前需要显示的ActivityStack。

6. 继续通过ApplicationThread来发送消息给主线程的Handler来启动Activity（handleLaunchActivity）。

7. handleLauchActivity：调用了performLauchActivity，里边Instrumentation生成了新的activity对象，继续调用activity生命周期。

#### IPC过程：

双方都是通过对方的代理对象来进行通信。  
1.app和AMS通信：app通过本进程的AMP和AMS进行Binder通信  
2.AMS和新app通信：通过ApplicationThreadProxy来通信，并不直接和ActivityThread通信



#### 参考函数流程

Activity启动流程（从Launcher开始）：

第一阶段： Launcher通知AMS要启动新的Activity（在Launcher所在的进程执行）

- Launcher.startActivitySafely //首先Launcher发起启动Activity的请求

- Activity.startActivity

- Activity.startActivityForResult

- Instrumentation.execStartActivity //交由Instrumentation代为发起请求

- ActivityManager.getService().startActivity //通过IActivityManagerSingleton.get()得到一个AMP代理对象

- ActivityManagerProxy.startActivity //通过AMP代理通知AMS启动activity

第二阶段：AMS先校验一下Activity的正确性，如果正确的话，会暂存一下Activity的信息。然后，AMS会通知Launcher程序pause Activity（在AMS所在进程执行）

- ActivityManagerService.startActivity

- ActivityManagerService.startActivityAsUser

- ActivityStackSupervisor.startActivityMayWait

- ActivityStackSupervisor.startActivityLocked ：检查有没有在AndroidManifest中注册

- ActivityStackSupervisor.startActivityUncheckedLocked

- ActivityStack.startActivityLocked ：判断是否需要创建一个新的任务来启动Activity。

- ActivityStack.resumeTopActivityLocked ：获取栈顶的activity，并通知Launcher应该pause掉这个Activity以便启动新的activity。

- ActivityStack.startPausingLocked

- ApplicationThreadProxy.schedulePauseActivity

第三阶段： pause Launcher的Activity，并通知AMS已经paused（在Launcher所在进程执行）

- ApplicationThread.schedulePauseActivity

- ActivityThread.queueOrSendMessage

- H.handleMessage

- ActivityThread.handlePauseActivity

- ActivityManagerProxy.activityPaused

第四阶段：检查activity所在进程是否存在，如果存在，就直接通知这个进程，在该进程中启动Activity；不存在的话，会调用Process.start创建一个新进程（执行在AMS进程）

- ActivityManagerService.activityPaused

- ActivityStack.activityPaused

- ActivityStack.completePauseLocked

- ActivityStack.resumeTopActivityLocked

- ActivityStack.startSpecificActivityLocked

- ActivityManagerService.startProcessLocked

- Process.start //在这里创建了新进程，新的进程会导入ActivityThread类，并执行它的main函数

第五阶段： 创建ActivityThread实例，执行一些初始化操作，并绑定Application。如果Application不存在，会调用LoadedApk.makeApplication创建一个新的Application对象。之后进入Loop循环。（执行在新创建的app进程）

- ActivityThread.main

- ActivityThread.attach(false) //声明不是系统进程

- ActivityManagerProxy.attachApplication

第六阶段：处理新的应用进程发出的创建进程完成的通信请求，并通知新应用程序进程启动目标Activity组件（执行在AMS进程）

- ActivityManagerService.attachApplication //AMS绑定本地ApplicationThread对象，后续通过ApplicationThreadProxy来通信。

- ActivityManagerService.attachApplicationLocked

- ActivityStack.realStartActivityLocked //真正要启动Activity了！

- ApplicationThreadProxy.scheduleLaunchActivity //AMS通过ATP通知app进程启动Activity

第七阶段： 加载MainActivity类，调用onCreate声明周期方法（执行在新启动的app进程）

- ApplicationThread.scheduleLaunchActivity //ApplicationThread发消息给AT

- ActivityThread.queueOrSendMessage

- H.handleMessage //AT的Handler来处理接收到的LAUNCH_ACTIVITY的消息

- ActivityThread.handleLaunchActivity

- ActivityThread.performLaunchActivity

- Instrumentation.newActivity //调用Instrumentation类来新建一个Activity对象

- Instrumentation.callActivityOnCreate

- MainActivity.onCreate

- ActivityThread.handleResumeActivity

- AMP.activityResumed

- AMS.activityResumed(AMS进程)




