---
layout:     post
title:      "Android进程间通信 - Messenger的使用和理解"
subtitle:   ""
date:       2019-06-25 11:36:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### Messenger简介

Messenger是基于Message对象进行跨进程通信的，类似于Handler发送消息实现线程间通信一样的用法。

### Messenger使用

下面写个客户端跨进程发送消息到服务端，服务端接受到立即回复的例子

### 服务端

实现流程

1. 首先创建一个Handler对象
2. 接着创建一个Messenger对象，并把Handler对象以形参传入Messenger中
3. 最后通过mMessenger.getBinder得到Binder对象，在onBind方法中返回。

```java
public class MessengerService extends Service {

    private static final String TAG = "MessengerService";
    public MessengerService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }

    private MessengerHandler mHandler=new MessengerHandler();

    private Messenger mMessenger=new Messenger(mHandler);

    private static class MessengerHandler extends Handler{

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            //取出客户端的消息内容
            Bundle bundle = msg.getData();
            String clientMsg = bundle.getString("client");
            Log.i(TAG,"来自客户端的消息："+clientMsg);
            //新建一个Message对象，作为回复客户端的对象
            Message message = Message.obtain();
            Bundle bundle1 = new Bundle();
            bundle1.putString("service","今天就不去了，还有很多东西要学！！");
            message.setData(bundle1);
            try {
                msg.replyTo.send(message);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
}
```

接受客户端的消息，其实与线程间通信一样，同样是在handleMessage方法中接受；如果服务端需要回复客户端，则需要拿到客户端携带过来的Messenger 对象（即**msg.replyTo**），通过msg.replyTo.send方法给客户端发送信息。

### 客户端

```java
public class MessengerActivity extends AppCompatActivity{

    private static final String TAG = "MessengerActivity";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);

    }


    public void onClick(View view) {
        switch (view.getId()){
            case R.id.bindService:
                Intent intent = new Intent(this,MessengerService.class);
                bindService(intent,mConnection,BIND_AUTO_CREATE);
                break;
            case R.id.unbindService:
                unbindService(mConnection);
                break;
        }
    }



    private ServiceConnection mConnection=new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //获取服务端关联的Messenger对象
            Messenger mService=new Messenger(service);
            //创建Message对象
            Message message = Message.obtain();
            Bundle bundle = new Bundle();
            bundle.putString("client","今天出去玩吗？");
            message.setData(bundle);
            //在message中添加一个回复mRelyMessenger对象
            message.replyTo=mRelyMessenger;
            try {
                mService.send(message);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
    };

    private GetRelyHandler mGetRelyHandler=new GetRelyHandler();

    private Messenger mRelyMessenger=new Messenger(mGetRelyHandler);
    public static class  GetRelyHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            Bundle bundle = msg.getData();
            String serviceMsg = bundle.getString("service");
            Log.i(TAG, "来自服务端的回复："+serviceMsg);

        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

    }

}
```

 客户端在绑定服务之后，在ServiceConnection中通过IBinder得到Messenger 对象（mService），然后用mService的send方法把Message作为形参发送给客户端。如果服务端收到消息需要回复客户端，同样的需要先创建一个Handler对象，接着创建一个Messenger对象（mRelyMessenger）并把Handler传入其中，最后在onServiceConnected方法中，把mRelyMessenger对象赋值给message.replyTo，和服务端的流程类似。

注意  
客户端和服务端中的msg.replyTo其实是同一个对象  
客户端首次发送消息是Message（mService）发送  
服务端发送消息是msg.replyTo发送。

### 注册文件

```java
   <service
            android:name=".messenger.MessengerService"
            android:enabled="true"
            android:exported="true"
            android:process=":remote">
        </service>
```

### 运行结果

首先是服务端收到客户端的消息，接着是客户端收到服务端的回复。

服务端接受的消息：

![](https://img-blog.csdn.net/20180718004022994?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h6dzIwMTc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

客户端接受服务端的回复消息：

![](https://img-blog.csdn.net/20180718004202449?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h6dzIwMTc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### Messenger理解

先了解服务端中onBind的返回值，mMessenger.getBinder()方法

```java
@Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
```

getBinder方法的源码

```java
/**
     * Retrieve the IBinder that this Messenger is using to communicate with
     * its associated Handler.
     * 
     * @return Returns the IBinder backing this Messenger.
     */
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
```

从源码可知，通过mTarget对象得到Messenger关联的IBinder对象，这就是Service中onBind方法需要的返回值，接着看看mTarget是怎么来的。

```java
  private final IMessenger mTarget;

    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
```

这不是我们创建成员变量mMessenger的构造方法吗，没错就是它**new Messenger(mHandler)**，接着getIMessenger方法的实现

```java
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}

private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}
```

 其实getIMessenger的返回值就是MessengerImpl 的实例，也就是说mTarget就是MessengerImpl对象，同时MessengerImpl是继承于IMessenger.Stub，看到这个是不是有点像aidl文件生成的java类，没错就是aidl生成的文件，看看源码就知道了。

```java
public interface IMessenger extends android.os.IInterface {
    /** Local-side IPC implementation stub class. */
    public static abstract class Stub extends android.os.Binder implements
            android.os.IMessenger {
        private static final java.lang.String DESCRIPTOR = "android.os.IMessenger";

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        public static android.os.IMessenger asInterface(...}

        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data,
                android.os.Parcel reply, int flags)
                throws android.os.RemoteException {...}

        private static class Proxy implements android.os.IMessenger {...}

    public void send(android.os.Message msg)
            throws android.os.RemoteException;
}
```

到这一步，算是明白了为什么可以mMessenger.getBinder()方法得到Binder对象了，最终还是AIDL文件来完成的，只不过Messenger进行了深度的封装AIDL，使用起来更简单。

#### 客户端

 客户端其实没什么，和平时使用一样的，首先通过onServiceConnected拿到sevice（Ibinder）对象，只不过得到的方式不一样

```java
Messenger mService=new Messenger(service);
```

这个aidl生成的java类是一样得，IMessenger.Stub是个Binder对象，通过Binder得到相关的公共接口IMessenger实例。

### 总结

![](https://img-blog.csdn.net/20180718021002659?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h6dzIwMTc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

相对AIDL来说，Messenger的使用是很简单了，省去中间很多繁琐的操作。对AIDL进行了封装，也就是对 Binder 的封装，是进程间通信成本较低的一种方式之一。
