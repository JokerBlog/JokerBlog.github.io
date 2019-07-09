---
layout:     post
title:      "JobScheduler的使用"
subtitle:   ""
date:       2019-07-09 10:55:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

也许我们可能会这么做: 注册一个广播接受者 , 监听屏幕熄灭状态 , 熄灭之后检查网络状态 , 然后再在广播接受者中启动一个服务去下载新的apk。可以是可以，但是这个会不会过于复杂了。而且如果出现需求变更......  
比如希望在特定情况下再启动事务，比如说延迟若干时间之后，或者等手机空闲了再运行，这样一方面不会在系统资源紧张之时喧宾夺主，另一方面也起到削峰填谷提高系统效率的作用。对于这些额外的条件要求，Service并不能直接支持，往往需要加入其他手段，才能较好地满足相关的运行条件

1. 对于延迟时间执行，通常考虑利用系统的闹钟管理器AlarmManager进行定时管理
2. 对于是否联网、是否充电、是否空闲，一般要监听系统的相应广播，常见的系统广播说明如下：

- 网络状态变化需要监听系统广播android.net.conn.CONNECTIVITY_CHANGE；
- 设备是否充电需要监听系统广播Intent.ACTION_POWER_CONNECTED也就是android.intent.action.ACTION_POWER_CONNECTED；
- 设备是否空闲需要监听系统广播Intent.ACTION_SCREEN_OFF也就是android.intent.action.SCREEN_OFF；

Android从5.0开始，增加支持一种特殊的机制，即任务调度JobScheduler，该工具集成了常见的几种运行条件，开发者只需添加少数几行代码，即可完成原来要多种组件配合的工作，使代码变得更加优（牛）雅（x）。



### 认识一下必须要用的工具

1. JobScheduler: 任务调度器
2. JobInfo : 任务概要信息
3. JobService: 任务服务，描述具体逻辑



### 简单使用JobScheduler

1. 创建JobService  

   我们具体的业务逻辑还是要写在jobService中的, 所以自定义一个服务继承自JobService 并重写两个抽象方法

`onStartJob`：在任务开始执行时触发。返回false表示执行完毕，返回true表示需要开发者自己调用jobFinished方法通知系统已执行完成。  
`onStopJob`，在任务停止执行时触发。

2. Activity中配置JobInfo  

   JobInfo是从来描述任务的执行时间，条件，策略等一系列的行为，使用Builder模式来获取实例，这里摘一下官方给出的代码

```java
JobInfo.Builder builder = new JobInfo.Builder(mJobId++, mServiceComponent);

        String delay = mDelayEditText.getText().toString();
        if (!TextUtils.isEmpty(delay)) {
            //设置至少延迟多久后执行，单位毫秒.  
            builder.setMinimumLatency(Long.valueOf(delay) * 1000);
        }
        String deadline = mDeadlineEditText.getText().toString();
        if (!TextUtils.isEmpty(deadline)) {
            //设置最多延迟多久后执行，单位毫秒。
            builder.setOverrideDeadline(Long.valueOf(deadline) * 1000);
        }
        boolean requiresUnmetered = mWiFiConnectivityRadioButton.isChecked();
        boolean requiresAnyConnectivity = mAnyConnectivityRadioButton.isChecked();
        if (requiresUnmetered) {
        //设置需要的网络条件，有三个取值：
        //JobInfo.NETWORK_TYPE_NONE（无网络时执行，默认）、
        //JobInfo.NETWORK_TYPE_ANY（有网络时执行）、
        //JobInfo.NETWORK_TYPE_UNMETERED（网络无需付费时执行） 
            builder.setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED);
        } else if (requiresAnyConnectivity) {  
            builder.setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY);
        }
        //是否在空闲时执行
        builder.setRequiresDeviceIdle(mRequiresIdleCheckbox.isChecked());
        
        //是否在充电时执行 
       builder.setRequiresCharging(mRequiresChargingCheckBox.isChecked());

        // Extras, work duration.
        PersistableBundle extras = new PersistableBundle();
        String workDuration = mDurationTimeEditText.getText().toString();
        if (TextUtils.isEmpty(workDuration)) {
            workDuration = "1";
        }
        extras.putLong(WORK_DURATION_KEY, Long.valueOf(workDuration) * 1000);

        builder.setExtras(extras);

        // Schedule job
        Log.d(TAG, "Scheduling job");
        JobScheduler tm = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
        tm.schedule(builder.build());
```

- etRequiredNetworkType：设置需要的网络条件，有三个取值：-  

  JobInfo.NETWORK_TYPE_NONE（无网络时执行，默认）、JobInfo.NETWORK_TYPE_ANY（有网络时执行）、JobInfo.NETWORK_TYPE_UNMETERED（网络无需付费时执行）
- setPersisted：重启后是否还要继续执行，此时需要声明权限RECEIVE_BOOT_COMPLETED，否则会报错“java.lang.IllegalArgumentException: Error: requested job be persisted without holding RECEIVE_BOOT_COMPLETED permission.”而且RECEIVE_BOOT_COMPLETED需要在安装的时候就要声明，如果一开始没声明，而在升级时才声明，那么依然会报权限不足的错误。
- setRequiresCharging：是否在充电时执行
- setRequiresDeviceIdle：是否在空闲时执行
- setPeriodic：设置时间间隔，单位毫秒。该方法不能和  

  setMinimumLatency、setOverrideDeadline这两个同时调用，否则会报错“java.lang.IllegalArgumentException: Can't call setMinimumLatency() on a periodic job”，或者报错“java.lang.IllegalArgumentException: Can't call setOverrideDeadline() on a periodic job”。
- setMinimumLatency：设置至少延迟多久后执行，单位毫秒。
- setOverrideDeadline：设置最多延迟多久后执行，单位毫秒。
- setBackoffCriteria: 退避策略 , 可以设置等待时间以及重连策略
- build：完成条件设置，返回构建好的JobInfo对象。

3. 声明MyJobService并将jobinfo加入, 执行  

   就是上面代码的最后两句了, 这里也可以看到JobScheduler 的本质其实就是系统的服务。



这里吧把官方demo中的JobService拿出来说一下吧，其大致流程就是我们使用JobScheduler的方法。它本质上是一个Service



```java
public class MyJobService extends JobService {

    private static final String TAG = MyJobService.class.getSimpleName();

    private Messenger mActivityMessenger;

    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(TAG, "Service created");
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.i(TAG, "Service destroyed");
    }

    /**
     * When the app's MainActivity is created, it starts this service. This is so that the
     * activity and this service can communicate back and forth. See "setUiCallback()"
     */
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        mActivityMessenger = intent.getParcelableExtra(MESSENGER_INTENT_KEY);
        return START_NOT_STICKY;
    }

    @Override
    public boolean onStartJob(final JobParameters params) {
        // The work that this service "does" is simply wait for a certain duration and finish
        // the job (on another thread).

        sendMessage(MSG_COLOR_START, params.getJobId());

        long duration = params.getExtras().getLong(WORK_DURATION_KEY);

        // Uses a handler to delay the execution of jobFinished().
        Handler handler = new Handler();
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                sendMessage(MSG_COLOR_STOP, params.getJobId());
                jobFinished(params, false);
            }
        }, duration);
        Log.i(TAG, "on start job: " + params.getJobId());

        // Return true as there's more work to be done with this job.
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        // Stop tracking these job parameters, as we've 'finished' executing.
        sendMessage(MSG_COLOR_STOP, params.getJobId());
        Log.i(TAG, "on stop job: " + params.getJobId());

        // Return false to drop the job.
        return false;
    }

    private void sendMessage(int messageID, @Nullable Object params) {
        // If this service is launched by the JobScheduler, there's no callback Messenger. It
        // only exists when the MainActivity calls startService() with the callback in the Intent.
        if (mActivityMessenger == null) {
            Log.d(TAG, "Service is bound, not started. There's no callback to send a message to.");
            return;
        }
        Message m = Message.obtain();
        m.what = messageID;
        m.obj = params;
        try {
            mActivityMessenger.send(m);
        } catch (RemoteException e) {
            Log.e(TAG, "Error passing service object back to activity.");
        }
    }
}
```

首先看必须重写的两个方法:  
onStartJob和onStopJob，

- 在onStartJob方法中使用Messager发送了一条消息，Tag为MSG_COLOR_START， 我们暂时先不管消息发送到了哪里，先宏观的看一下整个流程。然后发送了一个延时的消息MSG_COLOR_STOP，最后调用jobFinished方法结束。
- 在onStopJob方法中，直接发送了一条MSG_COLOR_STOP的消息

看来这里发送消息应该是重头戏了。  
首先回到Activity中，我们在onStart方法中找到了这样的代码

```java
Intent startServiceIntent = new Intent(this, MyJobService.class);
Messenger messengerIncoming = new Messenger(mHandler);
startServiceIntent.putExtra(MESSENGER_INTENT_KEY, messengerIncoming);
startService(startServiceIntent);
```

在Messager创建的时候就将一个handler作为参数来构造，之后作为extra来启动了服务。所以我们就能在服务中心的onStartCommand方法中获取到Messager的实例。接下来在服务的sendMessage方法中，也是直接用到了获取到的messager来发送消息，那肯定是发送到MainActivity传入的Handler中了，这里面的代码就不赘述了，都懂得。

#### 实现过程的源码

JobScheduler的创建, 是使用的getSystemService , 我们首先点开JobScheduler类 , 尼玛居然是一个抽象类? 喵喵喵?怎么办呢? 可以去翻一下getSystemService 方法 , emm你会很惊喜的  
既然是获取系统服, 按照规范, 其命名方式一定是xxxService , 所以我们直接去源码包下面搜JobScheduler, 看看会不会有什么发现

![](https://upload-images.jianshu.io/upload_images/6913461-d099c1a5438f1e15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/892/format/webp)

通过这种方式 , 我们可以找到获取到的真正的服务: JobSchedulerService  
但是问题来了, 我们的App是一个进程 , 系统服务是在系统的进程, 两者之间怎么进行通讯呢? 对了 , 进程间通讯 , 一定是用到了AIDL

继续在JobSchedulerService中找 , 在第762行会发现

```java
final class JobSchedulerStub extends IJobScheduler.Stub
```

那么根据AIDL的规范 , 一定有 IJobScheduler对应的.aidl文件  
打开这个文件 IJobScheduler.aidl :

```java
interface IJobScheduler {
    int schedule(in JobInfo job);
    void cancel(int jobId);
    void cancelAll();
    List<JobInfo> getAllPendingJobs();
}
```

定义了aidl之后会自动生成相应的接口类  
既然这个接口的存根(Stub)被继承 , 自然会重写接口中的方法 ,这些重写方法可以在JobSchedulerService$JobSchedulerStub中找到. 所以 , 我们在使用JobScheduler的一些列方法都应该是在这里了吧 , 但是JobScheduler和JobScheduler和JobSchedulerService是怎么关联在一起的呢?

因为JobScheduler是一个抽象类 , 必然有实现类 , 按照规范 , 明明应该为xxxImpl. 我们在之前的搜索中已经看到过它了 , 接下来点开看看详情

```java
public class JobSchedulerImpl extends JobScheduler
```

恩 , 没毛病就是它了 , 实现类当然会实现抽象方法中的抽象方法了  
在看构造方法 :

```java
IJobScheduler mBinder;

    /* package */ JobSchedulerImpl(IJobScheduler binder) {
        mBinder = binder;
    }
```

这里甚至直接命名为binder , 和我们绑定服务的套路一模一样:  
bindService中 , 重写onServiceConnected , 将传入的iBinder对象调用asInterface方法 ,就能获得aidl对应的接口实现 , 也就可以调用对应接口实现的方法.

再回到内部类JobSchedulerStub 中: 首先我们来看Scheduler方法

```java
@Override
        public int schedule(JobInfo job) throws RemoteException {
            if (DEBUG) {
                Slog.d(TAG, "Scheduling job: " + job.toString());
            }
            final int pid = Binder.getCallingPid();
            final int uid = Binder.getCallingUid();

            enforceValidJobRequest(uid, job);
            if (job.isPersisted()) {
                if (!canPersistJobs(pid, uid)) {
                    throw new IllegalArgumentException("Error: requested job be persisted without"
                            + " holding RECEIVE_BOOT_COMPLETED permission.");
                }
            }

            long ident = Binder.clearCallingIdentity();
            try {
                return JobSchedulerService.this.schedule(job, uid);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }
```

在针对pid和uid进行一系列的验证之后, 最终还是调用了外部类的scheduler方法:

```java
/**
     * Entry point from client to schedule the provided job.
     * This cancels the job if it's already been scheduled, and replaces it with the one provided.
     * @param job JobInfo object containing execution parameters
     * @param uId The package identifier of the application this job is for.
     * @return Result of this operation. See <code>JobScheduler#RESULT_*</code> return codes.
     */
public int schedule(JobInfo job, int uId) {
        JobStatus jobStatus = new JobStatus(job, uId);
        cancelJob(uId, job.getId());
        startTrackingJob(jobStatus);
        mHandler.obtainMessage(MSG_CHECK_JOB).sendToTarget();
        return JobScheduler.RESULT_SUCCESS;
    }
```

大家看下注释中的解释 , 大致了解一下流程  
这里重点说一下startTrackingJob方法

```java
private void startTrackingJob(JobStatus jobStatus) {
        boolean update;
        boolean rocking;
        synchronized (mJobs) {
            update = mJobs.add(jobStatus);
            rocking = mReadyToRock;
        }
        if (rocking) {
            for (int i=0; i<mControllers.size(); i++) {
                StateController controller = mControllers.get(i);
                if (update) {
                    controller.maybeStopTrackingJob(jobStatus);
                }
                controller.maybeStartTrackingJob(jobStatus);
            }
        }
    }
```

首先将jobStatus添加到JobStore中 , 根据是否添加成功 , 来决定是否执行  
controller.maybeStopTrackingJob(jobStatus); 这里也循环遍历了控制器 . 乍一看是不是懵逼了 , 什么狗东西?  
这里要提一下系统服务的创建过程了

Zygote--Linux的核心  
启动系统进程SystemServer时 , 就会创建一些关键的服务，比如AMS，PMS，WMS等，其中包括JobSchedulerService。我们来看一下SystemServer这个类，一共一千多行不是很难。

首先这是一个java程序，自然要先去找main函数，然后一步一步跟。  
在run方法中，我们可以看到一大堆的设置，往后直接找服务相关的（267行），三种服务：

```
startBootstrapServices();
startCoreServices();
startOtherServices();
```

我们在startOtherServices中可以发现,众多的服务都是通过SystemServiceManager去开启，它通过类名来反射构造器来实例化相应的Service，然后添加到系统服务集合中，最后启动服务。  
我们在开发中使用getSystemService的方式得到的服务，就是从刚刚提到了系统服务集合中获取的

既然是从**构造器**来实例化服务的，所以我们再回到JobSchedulerService看他的构造函数，发现在构造函数中，构造器的集合添加了一堆构造器，然后hander分发事件，读取本地的文件（data/system/job/jobs.xml）并执行任务。  
最终在maybeRunPendingJobsH()方法中,调用了executeRunnableJob方法，这个方法在JobServiceContext中。该方法中有onServiceConnected方法，即建立链接。同时也有很多很熟悉的方法，就不一一列举了。

在服务建立链接的同时，还进行防止睡眠等wakeLock操作，emmm，坏坏的。接下来使用Handler发送了消息，what值为**MSG_SERVICE_BOUND**, 跟进去方法, 最后我们还能看到service.startJob的操作, 即命令服务执行一个任务。

这样就将整个流程大致穿起来了，细节的地方大家自行查看吧，我觉得基本o98k，看源码看困了，睡会去- --








