---
layout:     post
title:      "android的消息处理机制（图+源码分析）——Looper,Handler,Message"
subtitle:   ""
date:       2019-07-11 09:02:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

      android的消息处理有三个核心类：Looper,Handler和Message。其实还有一个Message Queue（消息队列），但是MQ被封装到Looper里面了，我们不会直接与MQ打交道，因此我没将其作为核心类。下面一一介绍：



### 线程的魔法师 Looper

Looper的字面意思是“循环者”，它被设计用来使一个普通线程变成**Looper线程**。所谓Looper线程就是循环工作的线程。在程序开发中（尤其是GUI开发中），我们经常会需要一个线程不断循环，一旦有新任务则执行，执行完继续等待下一个任务，这就是Looper线程。使用Looper类创建Looper线程很简单：

```java
public class LooperThread extends Thread {
    @Override
    public void run() {
        // 将当前线程初始化为Looper线程
        Looper.prepare();
        
        // ...其他处理，如实例化handler
        
        // 开始循环处理消息队列
        Looper.loop();
    }
}
```

通过上面两行核心代码，你的线程就升级为Looper线程了！！！是不是很神奇？让我们放慢镜头，看看这两行代码各自做了什么。

#### 1)Looper.prepare()

![](https://pic002.cnblogs.com/images/2011/315542/2011091315284989.png)





通过上图可以看到，现在你的线程中有一个Looper对象，它的内部维护了一个消息队列MQ。注意，**一个Thread只能有一个Looper对象**，为什么呢？咱们来看源码。

```java
public class Looper {
    // 每个线程中的Looper对象其实是一个ThreadLocal，即线程本地存储(TLS)对象
    private static final ThreadLocal sThreadLocal = new ThreadLocal();
    // Looper内的消息队列
    final MessageQueue mQueue;
    // 当前线程
    Thread mThread;
    // 。。。其他属性

    // 每个Looper对象中有它的消息队列，和它所属的线程
    private Looper() {
        mQueue = new MessageQueue();
        mRun = true;
        mThread = Thread.currentThread();
    }

    // 我们调用该方法会在调用线程的TLS中创建Looper对象
    public static final void prepare() {
        if (sThreadLocal.get() != null) {
            // 试图在有Looper的线程中再次创建Looper将抛出异常
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper());
    }
    // 其他方法
}
```

通过源码，prepare()背后的工作方式一目了然，其核心就是将looper对象定义为ThreadLocal。如果你还不清楚什么是ThreadLocal，请参考[《理解ThreadLocal》](http://blog.csdn.net/qjyong/article/details/2158097)。



#### 2）Looper.loop()

![](https://pic002.cnblogs.com/images/2011/315542/2011091315590586.png)





调用loop方法后，Looper线程就开始真正工作了，它不断从自己的MQ中取出队头的消息(也叫任务)执行。其源码分析如下：

```java
  public static final void loop() {
        Looper me = myLooper();  //得到当前线程Looper
        MessageQueue queue = me.mQueue;  //得到当前looper的MQ
        
        // 这两行没看懂= = 不过不影响理解
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        // 开始循环
        while (true) {
            Message msg = queue.next(); // 取出message
            if (msg != null) {
                if (msg.target == null) {
                    // message没有target为结束信号，退出循环
                    return;
                }
                // 日志。。。
                if (me.mLogging!= null) me.mLogging.println(
                        ">>>>> Dispatching to " + msg.target + " "
                        + msg.callback + ": " + msg.what
                        );
                // 非常重要！将真正的处理工作交给message的target，即后面要讲的handler
                msg.target.dispatchMessage(msg);
                // 还是日志。。。
                if (me.mLogging!= null) me.mLogging.println(
                        "<<<<< Finished to    " + msg.target + " "
                        + msg.callback);
                
                // 下面没看懂，同样不影响理解
                final long newIdent = Binder.clearCallingIdentity();
                if (ident != newIdent) {
                    Log.wtf("Looper", "Thread identity changed from 0x"
                            + Long.toHexString(ident) + " to 0x"
                            + Long.toHexString(newIdent) + " while dispatching to "
                            + msg.target.getClass().getName() + " "
                            + msg.callback + " what=" + msg.what);
                }
                // 回收message资源
                msg.recycle();
            }
        }
    }
```

除了prepare()和loop()方法，Looper类还提供了一些有用的方法，比如

Looper.myLooper()得到当前线程looper对象：

```java
 public static final Looper myLooper() {
        // 在任意线程调用Looper.myLooper()返回的都是那个线程的looper
        return (Looper)sThreadLocal.get();
    }
```

getThread()得到looper对象所属线程：

```java
   public Thread getThread() {
        return mThread;
    }
```

quit()方法结束looper循环：

```java
   public void quit() {
        // 创建一个空的message，它的target为NULL，表示结束循环消息
        Message msg = Message.obtain();
        // 发出消息
        mQueue.enqueueMessage(msg, 0);
    }
```

到此为止，你应该对Looper有了基本的了解，总结几点：

1.每个线程有且最多只能有一个Looper对象，它是一个ThreadLocal

2.Looper内部有一个消息队列，loop()方法调用后线程开始不断从队列中取出消息执行

3.Looper使一个线程变成Looper线程。

那么，我们如何往MQ上添加消息呢？下面有请Handler！（掌声~~~）



### 异步处理大师 Handler

什么是handler？handler扮演了往MQ上添加消息和处理消息的角色（只处理由自己发出的消息），即**通知MQ它要执行一个任务(sendMessage)，并在loop到自己的时候执行该任务(handleMessage)，整个过程是异步的**。handler创建时会关联一个looper，默认的构造方法将关联当前线程的looper，不过这也是可以set的。默认的构造方法：

```java
public class handler {

    final MessageQueue mQueue;  // 关联的MQ
    final Looper mLooper;  // 关联的looper
    final Callback mCallback; 
    // 其他属性

    public Handler() {
        // 没看懂，直接略过，，，
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        // 默认将关联当前线程的looper
        mLooper = Looper.myLooper();
        // looper不能为空，即该默认的构造方法只能在looper线程中使用
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        // 重要！！！直接把关联looper的MQ作为自己的MQ，因此它的消息将发送到关联looper的MQ上
        mQueue = mLooper.mQueue;
        mCallback = null;
    }
    
    // 其他方法
}
```

下面我们就可以为之前的LooperThread类加入Handler：

```java
public class LooperThread extends Thread {
    private Handler handler1;
    private Handler handler2;

    @Override
    public void run() {
        // 将当前线程初始化为Looper线程
        Looper.prepare();
        
        // 实例化两个handler
        handler1 = new Handler();
        handler2 = new Handler();
        
        // 开始循环处理消息队列
        Looper.loop();
    }
}
```

加入handler后的效果如下图

![](https://pic002.cnblogs.com/images/2011/315542/2011091322483894.png)



可以看到，**一个线程可以有多个Handler，但是只能有一个Looper！**

**Handler发送消息**

有了handler之后，我们就可以使用 post(Runnable), postAtTime(Runnable, long), postDelayed(Runnable, long), sendEmptyMessage(int), sendMessage(Message), sendMessageAtTime(Message, long)和 sendMessageDelayed(Message, long)这些方法向MQ上发送消息了。光看这些API你可能会觉得handler能发两种消息，一种是Runnable对象，一种是message对象，这是直观的理解，但其实post发出的Runnable对象最后都被封装成message对象了，见源码：

```java
 // 此方法用于向关联的MQ上发送Runnable对象，它的run方法将在handler关联的looper线程中执行
    public final boolean post(Runnable r)
    {
       // 注意getPostMessage(r)将runnable封装成message
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

    private final Message getPostMessage(Runnable r) {
        Message m = Message.obtain();  //得到空的message
        m.callback = r;  //将runnable设为message的callback，
        return m;
    }

    public boolean sendMessageAtTime(Message msg, long uptimeMillis)
    {
        boolean sent = false;
        MessageQueue queue = mQueue;
        if (queue != null) {
            msg.target = this;  // message的target必须设为该handler！
            sent = queue.enqueueMessage(msg, uptimeMillis);
        }
        else {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
        }
        return sent;
    }
```

其他方法就不罗列了，总之通过handler发出的message有如下特点：

1.message.target为该handler对象，这确保了looper执行到该message时能找到处理它的handler，即loop()方法中的关键代码

```java
msg.target.dispatchMessage(msg);
```

2.post发出的message，其callback为Runnable对象



#### Handler处理消息

说完了消息的发送，再来看下handler如何处理消息。消息的处理是通过核心方法[dispatchMessage](http://developer.android.com/reference/android/os/Handler.html#dispatchMessage%28android.os.Message%29)([Message](http://developer.android.com/reference/android/os/Message.html)msg)与钩子方法[handleMessage](http://developer.android.com/reference/android/os/Handler.html#handleMessage%28android.os.Message%29)([Message](http://developer.android.com/reference/android/os/Message.html)msg)完成的，见源码

```java
   // 处理消息，该方法由looper调用
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            // 如果message设置了callback，即runnable消息，处理callback！
            handleCallback(msg);
        } else {
            // 如果handler本身设置了callback，则执行callback
            if (mCallback != null) {
                 /* 这种方法允许让activity等来实现Handler.Callback接口，避免了自己编写handler重写handleMessage方法。见http://alex-yang-xiansoftware-com.iteye.com/blog/850865 */
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            // 如果message没有callback，则调用handler的钩子方法handleMessage
            handleMessage(msg);
        }
    }
    
    // 处理runnable消息
    private final void handleCallback(Message message) {
        message.callback.run();  //直接调用run方法！
    }
    // 由子类实现的钩子方法
    public void handleMessage(Message msg) {
    }
```

可以看到，除了[handleMessage](http://developer.android.com/reference/android/os/Handler.html#handleMessage%28android.os.Message%29)([Message](http://developer.android.com/reference/android/os/Message.html)msg)和Runnable对象的run方法由开发者实现外（实现具体逻辑），handler的内部工作机制对开发者是透明的。这正是handler API设计的精妙之处！

#### Handler的用处

我在小标题中将handler描述为“异步处理大师”，这归功于Handler拥有下面两个重要的特点：

1.handler可以在**任意线程发送消息**，这些消息会被添加到关联的MQ上。

![](https://pic002.cnblogs.com/images/2011/315542/2011091409532564.png)





2.handler是在它**关联的looper线程中处理消息**的。

![](https://pic002.cnblogs.com/images/2011/315542/2011091409541179.png)





这就解决了android最经典的不能在其他非主线程中更新UI的问题。**android的主线程也是一个looper线程**(looper在android中运用很广)，我们在其中创建的handler默认将关联主线程MQ。因此，利用handler的一个solution就是在activity中创建handler并将其引用传递给worker thread，worker thread执行完任务后使用handler发送消息通知activity更新UI。(过程如图)

![](https://pic002.cnblogs.com/images/2011/315542/2011091323582123.png)



下面给出sample代码，仅供参考：

```java
public class TestDriverActivity extends Activity {
    private TextView textview;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        textview = (TextView) findViewById(R.id.textview);
        // 创建并启动工作线程
        Thread workerThread = new Thread(new SampleTask(new MyHandler()));
        workerThread.start();
    }
    
    public void appendText(String msg) {
        textview.setText(textview.getText() + "\n" + msg);
    }
    
    class MyHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            String result = msg.getData().getString("message");
            // 更新UI
            appendText(result);
        }
    }
}
```

```java
public class SampleTask implements Runnable {
    private static final String TAG = SampleTask.class.getSimpleName();
    Handler handler;
    
    public SampleTask(Handler handler) {
        super();
        this.handler = handler;
    }

    @Override
    public void run() {
        try {  // 模拟执行某项任务，下载等
            Thread.sleep(5000);
            // 任务完成后通知activity更新UI
            Message msg = prepareMessage("task completed!");
            // message将被添加到主线程的MQ中
            handler.sendMessage(msg);
        } catch (InterruptedException e) {
            Log.d(TAG, "interrupted!");
        }

    }

    private Message prepareMessage(String str) {
        Message result = handler.obtainMessage();
        Bundle data = new Bundle();
        data.putString("message", str);
        result.setData(data);
        return result;
    }

}
```

当然，handler能做的远远不仅如此，由于它能post Runnable对象，它还能与Looper配合实现经典的Pipeline Thread(流水线线程)模式。请参考此文[《Android Guts: Intro to Loopers and Handlers》](http://mindtherobot.com/blog/159/android-guts-intro-to-loopers-and-handlers/)



### 封装任务 Message

在整个消息处理机制中，message又叫task，封装了任务携带的信息和处理该任务的handler。message的用法比较简单，这里不做总结了。但是有这么几点需要注意（待补充）：

1.尽管Message有public的默认构造方法，但是你应该通过Message.obtain()来从消息池中获得空消息对象，以节省资源。

2.如果你的message只需要携带简单的int信息，请优先使用Message.arg1和Message.arg2来传递信息，这比用Bundle更省内存

3.擅用message.what来标识信息，以便用不同方式处理message。














