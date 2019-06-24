---
layout:     post
title:      "Java反射以及在Android中的特殊应用"
subtitle:   ""
date:       2019-06-24 14:05:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### 1、反射的定义以及组成

关于反射，一般书上的定义是这样的:JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；这种动态获取信息以及动态调用对象方法的功能称为java语言的反射机制，这几句解释说明了反射的作用，动态的跟类进行交互，比如获取隐藏属性，修改属性，获取对象，创建对象或者方法等等，总之就一句话:

反射是一种具有与类进行动态交互能力的一种机制 为什么要强调动态交互呢？

因为一般情况下都是动态加载，也就是在运行的时候才会加载，而不是在编译的时候，在需要的时候才进行加载获取，或者说你可以在任何时候加载一个不存在的类到内存中，然后进行各种交互,或者获取一个没有公开的类的所有信息，换句话说，开发者可以随时随意的利用反射的这种机制动态进行一些特殊的事情。



**反射的组成**

由于反射最终也必须有类参与，因此反射的组成一般有下面几个方面组成:

1.java.lang.Class.java：类对象；

2.java.lang.reflect.Constructor.java：类的构造器对象；

3.java.lang.reflect.Method.java：类的方法对象；

4.java.lang.reflect.Field.java：类的属性对象；



![](http://mmbiz.qpic.cn/mmbiz/MOu2ZNAwZwOeHq9X3jbmmcicAm819wO5oJeKdZJvPR6KU7iaqtpujiaxGUKicL5ytdI4m4niagU1L9oWNktaibIzcMNA/?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

根据虚拟机的工作原理,一般情况下，类需要经过:加载->验证->准备->解析->初始化->使用->卸载这个过程，如果需要反射的类没有在内存中，那么首先会经过加载这个过程，并在在内存中生成一个class对象，有了这个class对象的引用，就可以发挥开发者的想象力，做自己想做的事情了。

**反射的作用**

前面只是说了反射是一种具有与Java类进行动态交互能力的一种机制，在Java和Android开发中，一般情况下下面几种场景会用到反射机制.

● 需要访问隐藏属性或者调用方法改变程序原来的逻辑，这个在开发中很常见的，由于一些原因，系统并没有开放一些接口出来，这个时候利用反射是一个有效的解决方法

● 自定义注解，注解就是在运行时利用反射机制来获取的。

●在开发中动态加载类，比如在Android中的动态加载解决65k问题等等，模块化和插件化都离不开反射，离开了反射寸步难行。



**反射的工作原理**

我们知道，每个java文件最终都会被编译成一个.class文件，这些Class对象承载了这个类的所有信息，包括父类、接口、构造函数、方法、属性等，这些class文件在程序运行时会被ClassLoader加载到虚拟机中。

当一个类被加载以后，Java虚拟机就会在内存中自动产生一个Class对象,而我们一般情况下用new来创建对象，实际上本质都是一样的，只是这些底层原理对我们开发者透明罢了，我们前面说了，有了class对象的引用，就相当于有了Method,Field,Constructor的一切信息，在Java中，有了对象的引用就有了一切，剩下怎么发挥是开发者自己的想象力所能决定的了。



### *2*、反射的简单事例

```java

public class Student {
    private int age;//年龄
    private String name;//姓名
    private String address;//地址
     private static String sTest;
    public Student() {
         throw new IllegalAccessError("Access to default Constructor Error!");
    }
    private Student(int age, String name, String address) {
        this.age = age;
        this.name = name;
        this.address = address;
         sTest = "测试反射";
    }
    private int getAge() {
        return age;
    }
    private void setAge(int age) {
        this.age = age;
    }
    private String getName() {
        return name;
    }
    private void setName(String name) {
        this.name = name;
    }
    private String getAddress() {
        return address;
    }
    private void setAddress(String address) {
        this.address = address;
    }
    private static String getTest() {
        return sTest;
    }
}
```

在这里为了练习，刻意用了private来修饰成员变量和方法 下面代码用构造器，方法和属性和静态方法分别来获取一下：

```java
public class StudentClient {
    public static void main(String[] args) throws Exception{
        Class<?> clazz=Class.forName("ClassLoader.Student");
        Constructor constructors=clazz.getDeclaredConstructor(int.class,String.class,String.class);
        constructors.setAccessible(true);
        //利用构造器生成对象
        Object mStudent=constructors.newInstance(27,"小文","北京市海定区XX号");
        System.out.println(mStudent.toString());
        //获取隐藏的int属性
        Field mAgeField=clazz.getDeclaredField("age");
        mAgeField.setAccessible(true);
        int age= (int) mAgeField.get(mStudent);
        System.out.println("年龄为:"+age);
        //调用隐藏的方法
        Method getAddressMethod=clazz.getDeclaredMethod("getAge");
        getAddressMethod.setAccessible(true);
        int newage= (int) getAddressMethod.invoke(mStudent);
        System.out.println("年龄为:"+newage);
        //调用静态方法
        Method getTestMethod=clazz.getDeclaredMethod("getTest");
        getTestMethod.setAccessible(true);
        String result= (String) getTestMethod.invoke(null);
        System.out.println("调用静态方法:"+result);
    }
}
```



![](http://mmbiz.qpic.cn/mmbiz/MOu2ZNAwZwOeHq9X3jbmmcicAm819wO5oakkQWQlcF7DaedQJovpAE1Gsnt5dGJ54BiaJKMnyCn3v1HQ4NDTdLkw/?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



大家都看得懂，应该可以理解，有同学说不是有很多getDeclared***和get***的方法吗， 实际上都差不多的，只不过用的范围不一样而已。

getDeclared***获取的是仅限于本类的所有的不受访问限制的；

而get***获取的是包括父类的但仅限于public修饰符的。

Field和Method也是一样的道理，这个大家注意一下就好。

最后一个需要注意的是调用静态方法和调用实例方法有点区别，调用实例方法一定需要一个类的实例，而调用静态方法不需要实例的引用，其实这是JVM的在执行方法上的有所区别。



大家都看得懂，应该可以理解，有同学说不是有很多getDeclared***和get***的方法吗， 实际上都差不多的，只不过用的范围不一样而已。

getDeclared***获取的是仅限于本类的所有的不受访问限制的；

而get***获取的是包括父类的但仅限于public修饰符的。

Field和Method也是一样的道理，这个大家注意一下就好。

最后一个需要注意的是调用静态方法和调用实例方法有点区别，调用实例方法一定需要一个类的实例，而调用静态方法不需要实例的引用，其实这是JVM的在执行方法上的有所区别。



### 3、反射在Android框架层的应用

这是本文需要重点说明的，众所周知，Android中的FrameWork是用Java语言编写的，自然离不开一些反射的影子，而利用反射更是可以达到我们一些常规方法难于达到的目的，再者反射也是Java层中进行Hook的重要手段，目前的插件化更是大量利用反射。  

**首先提出需求:如何监控Activity的创建和启动过程？**有同学说了，我在Activity里面重写生命周期方法不就可以了吗？

实际上这个是达不到需求的，因为很简单，这些生命周期方法的调用是在创建和启动之后很久的事情了，里面的生命周期方法相对于整个Activity来说是比较后面的事情，要想解决这个问题，必须要知道Activity是怎么来，中间经过了哪个流程，最后去了哪里，只有明白了这些，才能知道在哪个阶段做哪些事情，我们知道，Activity的启动是一个IPC过程，也就是Binder机制，里面经过了本地进程->AMS进程-->再回到本地进程，下面是实例图:

![](http://mmbiz.qpic.cn/mmbiz/MOu2ZNAwZwOeHq9X3jbmmcicAm819wO5oICowtk6DUibiah1XAVl4MlJVL3uYu2uiaIEHarhwICgHsuSZtDThX3OQQ/?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Handler--->handleMessage(LAUNCH_ACTIVITY)--->handleLaunchActivity--->performLaunchActivity--->Instrumentation



图画的有些粗糙，大家将就看吧，从上面可以看到，Activity从本地到远程AMS以后，远程AMS只是做了权限以及属性的检查,然后再回到本地进程，这才开始真正的创建和检查，我们才代码来分析一下，涉及到的类有Handler以及ActivityThread和Instrumentation类,首先从远端进程回到本地进程之后，系统的Handler类H会发送一个消息:LAUNCH_ACTIVITY,代码如下:省略了一些非必要代码，不然篇幅太长,下面的代码都是在ActivityThread.java里面的





```java

public void handleMessage(Message msg) {
    if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
    switch (msg.what) {
        case LAUNCH_ACTIVITY: {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
            final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
            r.packageInfo = getPackageInfoNoCheck(
                    r.activityInfo.applicationInfo, r.compatInfo);
            handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        } 
```

随后调用了handleLaunchActivity方法，handleLaunchActivity方法里面又调用了 performLaunchActivity方法，代码如下:

```java

private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;
    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
    }
    // Make sure we are running with the most recent config.
    handleConfigurationChanged(null, null);
    if (localLOGV) Slog.v(
        TAG, "Handling launch of " + r);
    // Initialize before creating the activity
    WindowManagerGlobal.initialize();
    Activity a = performLaunchActivity(r, customIntent);
    ....
    }
```

在performLaunchActivity里面终于创建了Activity了，进入performLaunchActivity里面看看有一段非常核心的代码:

```java
Activity activity = null;
try {
//
    java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
    //通过mInstrumentation.newActivity()方法创建了Activity,mInstrumentation是Instrumentation类的实例
    ，对象的类为:Instrumentation.java
    activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
    StrictMode.incrementExpectedActivityCount(activity.getClass());
    r.intent.setExtrasClassLoader(cl);
    r.intent.prepareToEnterProcess();
    if (r.state != null) {
        r.state.setClassLoader(cl);
    }
} catch (Exception e) {
    if (!mInstrumentation.onException(activity, e)) {
        throw new RuntimeException(
            "Unable to instantiate activity " + component
            + ": " + e.toString(), e);
    }
}
```

我们现在已经知道了Activity的创建了，是由Instrumentation的newActivity()方法实现，我们看一下方法:

```java
public Activity newActivity(Class<?> clazz, Context context, 
        IBinder token, Application application, Intent intent, ActivityInfo info, 
        CharSequence title, Activity parent, String id,
        Object lastNonConfigurationInstance) throws InstantiationException, 
        IllegalAccessException {
    Activity activity = (Activity)clazz.newInstance();
    ActivityThread aThread = null;
    activity.attach(context, aThread, this, token, 0, application, intent,
            info, title, parent, id,
            (Activity.NonConfigurationInstances)lastNonConfigurationInstance,
            new Configuration(), null, null, null);
    return activity;
}
```

看到没，作为四大组件的Activity其实也是一个普通对象，也是由反射创建的，只不过由于加入了生命周期方法，才有组件这个活生生的对象存在， 所以说Android中反射无处不在，分析完了启动和创建的过程，回到刚才那个需求来说，如何监控Activity的启动和创建呢？

读者可以先自己想一下，首先启动是由Handler来发送消息，具体的在里面handlerMessage方法实现的， 这也是Handler里面的处理代码的顺序，如下代码:

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

大不了我们自己弄一个自定义的Handler.Callback接口，然后替换掉那个H类里面的处理接口，这样就可以监控Activity的启动了，好方法，我们来写一下代码:

```java
public static void hookHandler(Context context) throws Exception {
    Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
    Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
    currentActivityThreadMethod.setAccessible(true);
    //获取主线程对象
    Object activityThread = currentActivityThreadMethod.invoke(null);
    //获取mH字段
    Field mH = activityThreadClass.getDeclaredField("mH");
    mH.setAccessible(true);
    //获取Handler
    Handler handler = (Handler) mH.get(activityThread);
    //获取原始的mCallBack字段
    Field mCallBack = Handler.class.getDeclaredField("mCallback");
    mCallBack.setAccessible(true);
    //这里设置了我们自己实现了接口的CallBack对象
    mCallBack.set(handler, new UserHandler(handler));
}
```

```java
public class UserHandler  implements Callback {
    //这个100一般情况下最好也反射获取，当然了你也可以直接写死，跟系统的保持一致就好了
    public static final int LAUNCH_ACTIVITY = 100;
    private Handler origin;
    public UserHandler( Handler mHandler) {
        this.origin = mHandler;
    }
    @Override
    public boolean handleMessage(Message msg) {
        if (msg.what == LAUNCH_ACTIVITY) {
        //这样每次启动的时候可以做些额外的事情
            Log.d("[app]","做你想要的事情");
        }
        origin.handleMessage(msg);
        return false;
    }
}
```

好了，Activity的启动监控就这样了，一般写在application里面的attachBaseContext()方法里面，因为这个方法时机最早。 

好了，下面来说说Activity的创建的监控，前面我们知道了，Instrumentation的newActivity方法负责创建了Activity，那么突破口也就是在这里了，创建为我们自定义的Instrumentation，然后反射替换掉就好，同时重写newActivity方法，可以做些事情，比如记录时间之类，下面是代码:

```java
public static void hookInstrumentation() throws Exception{
    Class<?> activityThread=Class.forName("android.app.ActivityThread");
    Method currentActivityThread=activityThread.getDeclaredMethod("currentActivityThread");
    currentActivityThread.setAccessible(true);
    //获取主线程对象
    Object activityThreadObject=currentActivityThread.invoke(null);
    //获取Instrumentation字段
    Field mInstrumentation=activityThread.getDeclaredField("mInstrumentation");
    mInstrumentation.setAccessible(true);
    Instrumentation instrumentation= (Instrumentation) mInstrumentation.get(activityThreadObject);
    CustomInstrumentation customInstrumentation=new CustomInstrumentation(instrumentation);
    //替换掉原来的,就是把系统的instrumentation替换为自己的Instrumentation对象
    mInstrumentation.set(activityThreadObject,CustomInstrumentation);
    Log.d("[app]","Hook Instrumentation成功");
}
```

```java
public class CustomInstrumentation  extends Instrumentation{
    private Instrumentation base;
    public CustomInstrumentation(Instrumentation base) {
        this.base = base;
    }
    //重写创建Activity的方法
    @Override
    public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        Log.d("[app]","you are hook!,做自己想要的事情");
        Log.d("[app]","className="+className+" intent="+intent);
        return super.newActivity(cl, className, intent);
    }
}
```

同样在application的attachBaseContext注入就好，当然了，Instrumentation还有其他方法可以重写，大家可以去试一试，下面是运行的结果:

![](http://mmbiz.qpic.cn/mmbiz/MOu2ZNAwZwOeHq9X3jbmmcicAm819wO5oXk3H6icJPoPWAcubBhBMD0yo97N4JESuQnhNm2VLH19yMcHLDNNbvRA/?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

看到没，监控启动和创建都实现了，其实这里面也有好多扩展的，比如启动Activity的时候，Instrumentation一样是可以监控的，你懂的，再次重写方法，然后实现自己的逻辑，另外，small插件化框架就是Hook了Instrumentation来动态加载Activity的，大家有兴趣可以去看看，除了以上方法，还有很多方法可以用类似的手段去实现，大家一定要多练习，好记性不如烂笔头就是这个道理。

**使用反射需要注意的地方**

从前面可以看出，使用反射非常方便，而且在一些特定的场合下可以实现特别的需求，但是使用反射也是需要注意一下几点的:

- 反射最好是使用public修饰符的，其他修饰符有一定的兼容性风险，比如这个版本有，另外的版本可能没有。

- 大家都知道的Android开源代码引起的兼容性的问题,这是Android系统开源的最大的问题，特别是那些第三方的ROM，要慎用。

- 如果大量使用反射，在代码上需要优化封装，不然不好管理，写代码不仅仅是实现功能，还有维护性和可读性方法也需要加强，demo中可以直接这样粗糙些，在项目中还是需要好好组织封装下的。














