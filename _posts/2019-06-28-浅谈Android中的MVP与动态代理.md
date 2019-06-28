---
layout:     post
title:      "浅谈Android中的MVP与动态代理"
subtitle:   ""
date:       2019-06-28 09:10:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android   
  
---

### 前言

在Android开发平台上接触MVP足足算起来大概已经有一个年头左右。从最开始到现在经历的几个项目中我都采用了MVP架构作为底层框架，使得在view（Activity/Fragment）层中的业务调用逻辑分离到另外的presenter层中，让view层变得非常的轻量，并且不会出现非常复杂的逻辑以及难以阅读和理解的代码块，并且对于编写单元测试用例的实现也是非常的方便和快捷的。



### MVC简介

在讨论MVP之前我想先讨论一下Android传统开发中一直默认使用的MVC架构，还记得当初做的第一次项目就是基于MVC的。

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt5NO8hcr6DdneckEjQJicH4FebKyicok0bCqiaVPtG4QycNDafiatkE4trj6WYZE595onIBauMAbjSb1Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



MVC分为：Model（数据抽象）、View（视图）、Controller（控制器）的三层架构。接下来我们分别来一一解析每一层所对应的职责分别是什么。  

- View层：对应的则是Android中的layout文件夹中的xml文件，在启动Activity/Fragment的时候，都会加载一个R.layout.xxx的布局文件，使得在视图中显示出我们在xml中定义好的视图。
- Controller层：对应的则是Activity/Fragment。当Activity/Fragment加载了layout文件后，我们需要在Activity/Fragment中findViewById(int)去寻找到相对应的view，并对找到的view设置相应的属性以及监听器。而在设置view的属性之前，我们很有可能会先到model中请求一次数据，当数据回调回来后controller就会去更新view了。
- Model层：对应的则是一些DataSource以及DataBean的相关对象，这里的DataSource指的是数据的来源。一般数据的来源有2个主要的地方，一个是sqlite，一个是webservice，而我们习惯于将这两种数据的来源封装在一个repository中，对于调用者而言只需要调用repository中的一个获取接口来获取数据，但是这个数据是从内存中还是sqlite还是webservice来，我们都不得而知，从保护了调用实现的逻辑，分解相关的实现，达到调用者的极度简单与简洁，且在单元测试中测试接口也是非常方便的。

我们简单的了解了一下MVC的分层结构后，我们来更加详细的分析一下在Android中，这三层分别是如何相互调用与通信的。

首先是View：Activity_view.xml

```php
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/btn_hello_mvc"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:text="Hello MVC" />

</FrameLayout>
```

接下来是Controller：ControllerActivity.java

```java
// Controller
public class ControllerActivity extends Activity {

    private Button mBtn;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.Activity_view);

        // 在此处，controller调用并访问了view
        mBtn = (Button) findViewById(R.id.btn_hello_mvc);
        mBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // 对于这个 OnClickListener，是属于view的，它是view的监听器
                // 在这里，view直接访问了model
                String btnClickData = ModelDataSource.ins().getBtnClickData();
                Toast.makeText(ControllerActivity.this, btnClickData, Toast.LENGTH_SHORT).show();
            }
        });

        // 在此处controller调用了model
        String btnText = ModelDataSource.ins().getBtnText();

        // 在此处controller设置了view的属性
        mBtn.setText(btnText);
    }
}
```

最后则是Model:ModelDataSource.java

```java
// Model
public class ModelDataSource {

    private static ModelDataSource mInstance = null;

    public static ModelDataSource ins() {
        if (mInstance == null) {
            synchronized (ModelDataSource.class) {
                if (mInstance == null) {
                    mInstance = new ModelDataSource();
                }
            }
        }
        return mInstance;
    }

    private ModelDataSource() {
    }

    public String getBtnText() {
        // 在这里，
        // 我们可以去数据库中查找数据，
        // 也可以去网络中获取数据
        return "I am from ModelDataSource";
    }

    public String getBtnClickData() {
        // 在这里，
        // 我们可以去数据库中查找数据，
        // 也可以去网络中获取数据
        return "Hello MVC!";
    }
}
```

在这里，我将model设置为了单例模式，我之所以采用单例，是因为model主要关注的是数据源，而整个模块的数据应该是保证数据的唯一性，这样无论在任何一处修改数据的时候，都可以在每一处都达到数据的统一性，从而保证了数据的安全。就好像一个账号对应的是一份密码一样。而DataBean则是简单的String类型了，我并没有去定义一个数据结构。

而在controller中，它的职责逻辑相对的复杂，它对于view需要将从model中获取而来的数据进行及时的呈现在ui上；而对于model而言controller将会依据app生命周期的变化对model的数据进行及时的刷新和获取，比如当我们接受到一个切换壁纸的广播提醒的时候，此时我们需要在controller中通过调用model来获取新的壁纸数据，然后更新到某处的缓存对象中，再由缓存对象发布出订阅，因为一个app有可能在多个地方需要监听壁纸的变动，例如项目C的icon和locker组件的预览界面，在两个不同的Fragment中需要同时监听壁纸的改变，为了更及时的更新到视图ui上。而在这个demo中，我只是在onCreate(bundle)的时候从model中获取了初始的数据然后更新到btn中并没有做过多通信，但即便如此也可以很直接的看出controller会因为生命周期的变化对model的数据进行良好的CRUD。

最后一个model层很多人会理解为是普通的javabean以及我的大学老师也是这么和我说的，但是我并不这么认为，我不认为model只是很简单的一个数据结构定义，更多的它应该包含大量的数据处理和运算的逻辑，例如从数据库中采集数据的操作或者通过网络请求或者通过NetStream的方法来获取到二进制的数据，接着将这些二进制转换为我们设定好的javabean也就是我们定义好的抽象数据模型，然后该对象进行传递以及显示到视图ui上。具体的model架构逻辑我希望下次在做更加详细讨论。

Demo运行结果：

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt5NO8hcr6DdneckEjQJicH4Fgozsz4K4lEBKxCLaUeTqnsSqmFqXHI9OcGa8wgzVeib9QKsc2SiaTnyQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



至此，我们大概简单的介绍完了MVC接下来我们用时序图的方式来做一个总结：

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt5NO8hcr6DdneckEjQJicH4FT4Jtu9fGIMEI1rLQnjz7Pshibgib7w1icIcypFqGrH5qROxjfhPcibrGpA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



通过时序图我们可以大致的看出整个过程中主要的依赖在与controller，controller不止要处理ui的呈现与事件的响应并且还需要负责和model的通信，且view层也会与model之间通信，三者之间强强关联。

最后我们给出它的优缺点：

- 优点：Android开发中默认使用的框架，易于上手，能在不需要考虑太多需求的情况下快速开发一些小型demo功能app。

- 缺点：随着业务的扩展controller会变的越来越臃肿和复杂，大大增加了开发人员的维护成本以及交接成本，使得后期工作难以展开，且随着逻辑的复杂变化以及时间的推移会出现连开发人员自身都对当前代码逻辑的复杂造成错误的理解。

在这里由于是demo所以Controller的代码并不是很多，但是放入项目中假设这是一个非常复杂的view，例如浏览器，它不只要处理顶部的温度天气的ui显示和业务逻辑还要处理底部的数据流以及上下滑动交互搜索等的一切ui显示和业务逻辑，如果将这一切的逻辑和呈现都按照MVC来设计，那在整个Activity中的代码逻辑将是异常的复杂和混乱并且会使得Activity异常臃肿，使得开发难度急剧上升在项目交接过程中也是需要耗费巨大的成本的，同时维护的成本也是巨大的，当然我这里只是假设也许事实并非如此哈。



### MVP简介

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt5NO8hcr6DdneckEjQJicH4FGfKWMic5oR18pVoY63uU8ypqQwOqdf1OuIfVnLRcgDzpEto5UiaPB6Zg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



从上图中我们可以很清晰的看到MVP与MVC中的区别：

- 从Controller变成了Presenter

- 去除View和Model之间的调用关系，从而彻底的分离了Model和View之间的关联与耦合

> MVP和MVC中更具体的区别我们放到后面在做总结与讨论，这里只是大概指出他们两者之间的不同之处。

还是老规矩，我们分别来介绍一下MVP架构中的：Model（数据模型）、View（视图）、Presenter（主持者）他们三者的职责以及相互之间的关系到底是如何运作的。

- View层:视图层，它所对应的不只是layout中的xml文件还包括了Activity/Fragment作为视图的显示。这样做是扩大了View层的职责所在，View不仅是设置ui的显示和属性并且还包括了生命周期的回调。

- Presenter层:主持者层，它相当于是Controller中的业务逻辑部分，它主要是负责view和model层之间的通信，及时的响应view层的请求并主动的调用model层的数据获取，并且将获取到的数据结果返回给view层中。presenter是另外新建立一个class，并且让view从创建的时候就持有一个presenter的实例，当view发生某些请求响应或者生命周期发生变化，则会迅速的向presenter发起请求，让presenter做出响应的处理，比如：刷新数据、清除数据防止泄露等。



- Model层：此处的数据抽象层model和MVC中的model层是一样的，这里就不做更多的叙述。

在MVP的架构中，有一个非常大的特点就是view和model之间的通信必须是通过presenter的传递，也正是因为这种隔离的关系，使得视图和数据之间的关系变得完全分离。当视图改变的时候，数据源部分的代码无需任何变动；而当数据源发生改变的时候，视图部分也根本无需替换。但是事实并非我描述的如此容易，只是在面对整个项目工程的改动来说，我们只需要修改model并且对view层毫无影响，尽管如此工作量依然不容小视。但是如果是使用mvc的默认构建，则会发现整个程序中几乎处处与model耦合，视图或交互的替换基本就是对整个项目的重构，成本是相当大的。

在view和presenter两者之间的通信并不是想怎么调用就可以怎么调用的，他们之间有着一个标准的协议，就是在两者之间定义通用接口IContract，在这个interfac中定义了view层中要暴露的接口也定义了presenter层中需要暴露给view的接口，其目的是利用接口的方式将两者进行隔离，两者之间谁都不认识谁的实现，达到面向接口编程的目的。接下来我们通过代码的形式来一起探索MVP在实际代码中是如何构建的。

View和Presenter之间的协议IContract.java

```java
// Contract
public interface IContract {
    interface View {

        void updateBtnText(String s);

        void showToast(String s);
    }

    interface Presenter {

        /**
         * 调用该方法表示presenter被激活了
         */
        void start();

        void loadClickString();

        /**
         * 调用此方法表示presenter要结束了
         * 其目的是为了接触相互持有导致的内存泄露
         */
        void destroy();
    }
}
```

View层ViewActivity.java：

```java
// View
public class ViewActivity extends Activity implements IContract.View {

    private Button mBtn;
    private IContract.Presenter mPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.Activity_view);

        // 在最开始的时候构建presenter
        mPresenter = new Presenter(this);

        // View初始化
        mBtn = (Button) findViewById(R.id.btn_hello_mvp);
        mBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                mPresenter.loadClickString();
            }
        });
    }

    @Override
    protected void onStart() {
        super.onStart();
        mPresenter.start();
    }

    @Override
    protected void onDestroy() {
        if (mPresenter != null) {
            mPresenter.destroy();
            mPresenter = null;
        }
        super.onDestroy();
    }

    @Override
    public void updateBtnText(String s) {
        mBtn.setText(s);
    }

    @Override
    public void showToast(String s) {
        Toast.makeText(this, s, Toast.LENGTH_SHORT).show();
    }
}
```

Presenter层Presenter.java:

```java
// Presenter
class Presenter implements IContract.Presenter {

    private IContract.View mView;

    Presenter(IContract.View view) {
        mView = view;
    }

    @Override
    public void start() {
        String s = ModelDataSource.ins().getBtnText();
        mView.updateBtnText(s);
    }

    @Override
    public void loadClickString() {
        String s = ModelDataSource.ins().getBtnClickData();
        mView.showToast(s);
    }

    @Override
    public void destroy() {
        mView = null;
    }
}
```

Model层ModelDataSource.java:

```java
// Model
public class ModelDataSource {

    private static ModelDataSource mInstance = null;

    public static ModelDataSource ins() {
        if (mInstance == null) {
            synchronized (ModelDataSource.class) {
                if (mInstance == null) {
                    mInstance = new ModelDataSource();
                }
            }
        }
        return mInstance;
    }

    public String getBtnText() {
        // 在这里，
        // 我们可以去数据库中查找数据，
        // 也可以去网络中获取数据
        return "I am from ModelDataSource";
    }

    public String getBtnClickData() {
        // 在这里，
        // 我们可以去数据库中查找数据，
        // 也可以去网络中获取数据
        return "Hello MVP!";
    }
}
```

> ModelDataSource.java的代码是参考MVC中的Model。 Model层中的代码和MVC中Model层的代码基本一致，无非就是改变getBtnClickData()中返回的数据，但这并不会影响到我们对MVP框架的认识。

从代码上看我们可以发现比起传统的MVC从代码数量上看似乎并没有减少反而增加了不少的代码和接口，从逻辑上看似乎有些晕乎。但事实并非如此，当我们理解了MVP后则会发现这种调用方式其实是非常清晰的，因为你根本无需去在乎到底是谁在调用你，你只需要知道：我要让M做什么并且当M做完后我需要将M得出的结果告诉指定的V即可。同时在逻辑上的理解也是非常容易的。

在ViewActivity中实现了IContract.View接口，并实现了updateBtnText()和showToast()这两个方法，但是这两个方法貌似都没有被调用，只是在onCreate()的时候创建了一个presenter对象，在onStart()的时候调用了presenter.start()方法，然后在onDestroy()的时候调用了presenter.destroy()方法，而当onClick事件响应的时候也调用了presenter.loadClickString()方法，那么既没有回调也没有直接调用，那view中的两个接口方法又是何时被响应的呢？接下来我们将继续分析presenter层的逻辑结构。

在Presenter中实现了IContract.Presenter接口并实现了start()\loadClickString()\destroy()方法，在构造方法中有一个view的参数，而这个对象则是view的引用，但是这个view到底是Activity还是Fragment又或者是任意一个接口的具体实现类都有可能，但对于p而言具体的view到底是谁并不知道。presenter和View有一个共同的特点，就是方法之间彼此并不会相互调用而是各自独立的存在。但是值得发现的一点是在start()和loadClickString()方法中除了调用model外都调用了view的方法：mView.updateBtnText()和mView.showToast()，以此来对view视图的ui呈现以及交互提醒做出相应的响应。而最后的destroy()方法则是用与释放对view的循环引用资源的。

由此我们可以得出一个结论： 

对于view来说：

- 我需要一个主持者，当出现view事件的响应或者生命周期的变化时，我需要告诉这位主持，我要做些什么。

- 我会提供一系列通用接口，以便于当主持完成我的请求后，调用相应的接口让我明白这件事的结论是如何。

- 我所有的请求都发给主持，让他帮我做决定，但是这件事的决定是如何做，我并不知道，但我需要结果。

对于presenter来说：

- 我只会接收到请求后找model寻求帮助，等model做完事情后通知我了，我在把结果传递给view。

- 我只知道指挥model做事、让view显示数据，但我不干活。

- 我相当于一座桥，连接着view和model这两座岛，他们谁也不认识谁，想要通信必须要通过我，如果没有我，他们两永远都不会认识。

接下来我们用时序图的方式更加清晰的认识MVP架构之间的调用关系：

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt5NO8hcr6DdneckEjQJicH4FYPVoXic74ZGpSNMib2EdNPEib76ChNRl9qmoibAHf7FtHQic3w94Bn5icsfg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

很显然，从时序图上我们可以看出其中的调用关系以及调用逻辑非常的清晰，并不会出现任何的跨道调用的现象，程序的执行过程是非常有条理性。 

因为有Presenter这个角色的存在使得view部分的代码看上去是非常的清晰的，每一个方法都有它自己的主要倾向和职责所在，彼此之间并不会相互耦合。而Presenter中的代码也是如此，每一个方法都只处理一件事，并不会做其他无相关的事情。

接下来我们再来讨论一下为什么View和Presenter之间需要一个IContract这样的接口角色存在，它存在的意义到底是什么呢？

我们回过头来继续观察ViewActivity.java和Presenter.java这两个类，他们都实现了IContract中的View接口和Presenter接口，而在Presenter中并没有直接对ViewActivity直接持有，而是持有了IContract.View 这样的一个对象；在ViewActivity中也是如此，成员变量的持有类型也是IContract.Presenter。也就是说其实View不一定是Activity，而Presenter也不一定就是Presenter.java，两者只需要是有实现IContract接口中的具体类即可。那么这个时候就有一个非常棒的事情可以做了——是单元测试。

- 单元测试：此时我们只需要建立一个额外的测试类，让这个测试类实现IContract.View接口，接着再将其传入到Presenter中，此时便自由的测试Presenter中的接口是否有效，是否在回调回我们相应的接口方法，回调方法的时候是否有给出我们想要的结果。而对于View层的单元测试也是如此，构建一个测试类并实现Presenter中所有的接口即可对View中的方法进行大力的测试，看看是否能达到我们想要的预期。

- 变更逻辑：业务逻辑的变更这个例子在项目B中得到了巨大的便捷与证实。我们都很清晰的明白在项目B中有 联系人、短信、通话记录等备份功能，但是这几个功能中仔细的观察可以发现它们的View层是完全一样的。那面对这样的需求来说，难道要copy几份相同的Activity代码然后通过修改不同的字符串来达到实现界面与功能之间的管理吗？我的给的答案是NO。然而此时的MVP则可以做到共享一个View，却达到不同的业务实现。

对此我们应该如何来实现呢？

1. 首先构建一个ViewActivity设计好视图UI布局，并实现好IContract中View的接口；

2. 接着分别为 联系人、短息、通话记录等功能构建不同的Presenter对象并都实现IContract中Presenter接口；

3. 然后在View构建Presenter对象的时候，根据外部传递来不同的参数值创建出不同的Presenter；

4. 最后我们的View中的Title文本以及要显示的文案都放在Presenter的start()方法中一一回调给View。

很显然，通过这种方式，我们并不需要对View做更多的改动，只需要更具不同的业务构建不同逻辑的Presenter给View即可，View则会按照生命周期和事件响应的方式通知给Presenter，而不同的Presenter则会做出不同的逻辑处理，这样就达到了View层的强大复用且在新增备份功能的时候达到开闭原则，只需要增加响相应的Presenter即可，而不是去大量的构建新的Activity或者写很多重复的code。同样的如果是View层发生变化那么我们只需要修改一个地方即可达到修改全部View的视图方案了。这是不是一件让人觉得很有趣的事呢？



其实针对这类的需求来说，构建多个Presenter并不是最优的解决方案，还有更有趣的方式来实现，但是这里不做过多的讨论。

至此，MVP的基本知识也介绍的差不多了，接下来我们一起来做个优缺点的总结：

- **优点**

使用MVP可达到低耦合高内聚并且尽可能的保证了开闭原则，非常符合当前的软件工程；由于模块间的耦合很小，可做并行开发，一边开发View，一边开发Model；适合大部分的App，代码逻辑清晰易懂，大大降低开发、维护和交接成本；视图和底层进行彻底的分离，View发生改变则只需要修改View部分代码，底层数据实现发生改变则只需要修改底层Model的代码。

- **缺点**

对于很小的demo来说构建复杂和麻烦，不适合短期、小型且以后不在做任何维护的模块开发。

> MVP在Android开发上虽然是一款非常不错的架构，但它并非万能，并不是所有的APP都适合MVP；与此同时MVP的变种也是非常多，但对于基础的MVP三种角色是必不可少的。Google官方有推出一些关于MVP的Demo：Google MVP Demo 有兴趣可以参考一下





### 项目A中的MVP

回忆起曾开发项目A的日子，依稀的记得那时大概是去年的国庆。在16年9月份的时候我和一位同学交流的时候无意中我们谈起了软件架构，那时候他和我说正在学习MVP，当时的我并不知道MVP到底是什么，对整个软件架构的认知只是停留在对设计模式并不怎么理解并且只知道新建class就开始coding并且完全不知道该何时合适的引入一些经典的设计使得整个软件体系结构变得更加有趣，以至于后面回过头看看自己曾经写的代码竟然是基本都不认识了。。。

先是听到有MVP这么一回事，接着凭借着好奇心我便开始在网络上寻找一些相关的资料。网络上的资料还是非常多的，五花八门，各有个的路子。看了一段时间的资料后，便开始想着寻找一些比较直接的Demo例子来帮助自己理解，于是就在github上看了Google官方的MVP架构tododemo，接着有在自己的电脑上写了一些MVP的相关Demo。

学习的本质是知行合一，大概过了一个月到了10月份，接手了项目A这个项目，在项目开始前我并不打算和之前一样埋头苦干，我希望能加入更多的设计与架构，做的能比以往做的更好，于是我便带着勇气将新学的MVP架构引进了项目A。

项目A中使用的MVP和前面介绍的差不多，是属于基础型MVP其特征主要表现在：

- 当Presenter中出现异步获取数据的时候回调回来的数据需要被更新到View上的时候，此时View可能已经消失了。

- 当Presenter中在做异步耗时操作的时候，如果View没能及时释放，很大概率的出现context泄露

- 极度容易NullPointerException

在Presenter中的大概实现我们以代码的形式来描述：

```java
@Override
public void requestLogin(String id, String pwd) {
    if (TextUtils.isEmpty(id) || TextUtils.isEmpty(pwd)) {
        mView.showToast(R.string.empty);
        return;
    }

    AccountSource.ins().login(id, pwd, new AccountSource.Callback() {

        @Override
        public void success() {
            if (mView == null) {
                return;
            }
            mView.showToast(R.string.login_success);
        }

        @Override
        public void failed() {
            if (mView == null) {
                return;
            }
            mView.showToast(R.string.login_failed);
        }
    });
}
```

在这里我只是放入了Presenter中的某一个函数的实现，因为View已经其他地方的  

实现方式是一致的，而Model中的实现在这里并不重要，我们只需要探讨MVP在项目A中的特征应用即可。

由上面的代码可以看出，在Presenter中做异步回调的时候，我们务必判断mView是否还存在，否则就会出现大批量的NullPointerException()。这是一种很简单且基础的方式来处理回调事件，在项目B中这种方案将被新增的一个ViewModel对象替代。



### 项目B中的MVP（引入抽象视图模型）

在项目B中我只要接管的是相册备份，对于相册备份这个功能来说是一个非常具有挑战性的任务，它所需要考虑的不只是图片的上传与下载更多的应该是如何与服务器做好数据相关的同步通信，客户端的数据必须时刻保持最新的状态并且要做到”我“比服务端更早的知道这张图片是否需要备份。

一般在使用MVP的时候我们通常都会为了解决异步回调和context内存泄露做很多功课。在网络上有不少的解决方案例如通过Loader的方式来加载Presenter，但其本质是为了延长Presenter的生命周期，使得Presenter能在View消亡后还持续存在。而本次我介绍的是将视图抽象成模型，使得数据构建在模型上，然后再更新到真实的视图UI中，其目的也是延长了Presenter的生命周期并且解决了Context相关的内存泄露问题。



根据本节重点，我们先构建一个PersonInfo的ViewModel，其代码如下：

```java
public class PersonInfoViewModel {
    String imgUrl; // 头像图片链接
    String name;   // 名字
    boolean sex;   // 性别
    int age;       // 年龄
}
```

看到这里你可能会觉得奇怪：这不就是普通的bean吗？对，没错这就是普通的bean，但是不同的意图是这个bean中的所有字段都是在视图UI中有一一对应的，如图：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt5NO8hcr6DdneckEjQJicH4FxLYicyCNqmlLIYw9ueEMiaEtFHnIdhDicTLpmAnsBgOypS0uuxtzIxUdw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

由上图我们可以很直接的看到四个字段分别对应四个UI，那么我们只需要做到让Presenter更新数据到VM对象，接着在VM对象中排查View对象是否存在，如果活着就更新到View否则就不更新即可。

如此说来我们只需要以下这3步：

1. 让VM变为可订阅对象，当VM对象发现改变通知到View更新。

2. 分离View和Presenter，Presenter的数据只需要更新到VM即可。

3. 将VM对象与View对象链接。

为了达到我们想要的效果，接下来我们需要重新设计一下VM对象了：

```java
public class PersonInfoViewModel {

    String imgUrl;
    String name;
    boolean sex;
    int age;

    private static PersonInfoViewModel mInstance = null;

    public static PersonInfoViewModel getInstance() {
        if (mInstance == null) {
            synchronized (PersonInfoViewModel.class) {
                if (mInstance == null) {
                    mInstance = new PersonInfoViewModel();
                }
            }
        }
        return mInstance;
    }

    interface IOnDataChange {
        void onChange(PersonInfoViewModel viewModel);
    }

    private IOnDataChange mView;

    public void bind(IOnDataChange view) {
        mView = view;
        notifyDataSetChange();
    }

    public void unbind() {
        mView = null;
    }

    public void notifyDataSetChange() {
        if (mView != null) {
            mView.onChange(this);
        }
    }
}
```

在这里，我们将视图抽象模型设置为单例模式，因为视图抽象模型毕竟还是装数据的  
集合，而数据在全局中应该是保证同步和精准的，所以一般情况下是单一的。就好像  
一个app一般情况下是一份数据库不会同时跑多份数据库来存储相同的数据。

由以上代码中我们可以发现，VM成为了可订阅对象使得在View.onCreate()的时候订阅在View.onDestroy()的时候销毁，即使数据一直延迟回来也不会干扰到View的释放与泄露了。

接着我们再来看看Presenter中的异步实现：

```java
@Override
public void refreshPersonInfo(String token) {
    if (TextUtils.isEmpty(token)) {
        mView.showToast(R.string.empty);
        return;
    }

    AccountSource.ins().presonInfo(token, new AccountSource.Callback() {

        @Override
        public void success(String infoJson) {
            PersonInfoViewModel model = new Gson().fromJson(infoJson, PersonInfoViewModel.class);
            PersonInfoViewModel instance = PersonInfoViewModel.getInstance();
            instance.age = model.age;
            instance.imgUrl = model.imgUrl;
            instance.name = model.name;
            instance.sex = model.sex;
            instance.notifyDataSetChange();
        }

        @Override
        public void failed() {
        }
    });
}
```

由于和View的分离只需要更新数据，我们就可以很直接很简单的将数据更新到VM中，如果视图有和VM绑定那么一定会同步到View，如果没有则会一直存活在缓存中，等待下次View的bind()事件触发的时候再将数据寄回到View中。

接下来我们再来看看View中的实现是如何的：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    PersonInfoViewModel.getInstance().bind(this);
}

@Override
protected void onDestroy() {
    PersonInfoViewModel.getInstance().unbind();
    super.onDestroy();
}

@Override
public void onChange(PersonInfoViewModel viewModel) {
    mName.setText(viewModel.name);
    mSex.setText(viewModel.sex ? "男" : "女");
    mAge.setText(String.valueOf(viewModel.age));
    Glide.with(this).load(viewModel.imgUrl).into(mImg);
}
```

这样我们就能在第一时间更新数据到UI上了，是不是比之前更加的灵活呢？   
但这依然是存在缺陷与不足的：

- 如果要想在Presenter中通知View弹出Toast，Dialog时，该怎么办？

- 如果我们当前这个View是一个非常复杂的View，那onChange()方法岂不是变得异常复杂了？

- 如果我们只想更新局部的View数据，那该怎么办？

等等一堆的问题让我又从MVPVM倒回了MVP，Presenter依然需要持有View，但不是真的View而是一个假货。具体该如何实现呢？这也是我们需要在下一节 项目C中的MVP 中所要讨论的方法了。



### 项目C中的MVP（引入视图代理+一级缓存方法）

项目C是我接手的大概算是第5个项目了。在我刚接触到项目C的代码时，我发现项目C的代码竟然也是跑MVP的，我顿时刚到无比的喜悦与惊喜。但待我深入查看后发现虽然使用了MVP但是却依然是停留在最初的阶段。不过这并没有什么，我决心将其改变使得整体的架构变得更加有趣。在项目C中引入了RxJava2和Retrofit2，这两项优秀的开源库是我一直渴望学习和使用的，但遗憾的是一直未能找到时间。刚好此次项目C引入了这两项库的结合使用，使得我更加的兴奋与激动。

接下来在讨论本节内容前，需要先了解一些相关知识：

动态代理现在Android开发中广泛的被使用着，包括插件化、retrofit2等新技术都会使用着动态代理。通过动态代理的方式，我们可以在对业务完全透明的情况下去修改方法的执行过程。在这里我们就不做更多的讨论了。

本节在MVP中引入的视图代理和一级缓存方法都是依赖于动态代理实现。原本是通过静态代理实现，但是在实践的过程中我们很快就发现静态代理是符合OOP设计的，但是代码冗余太厉害且太过于繁琐。于是我们发现了动态代理机制。动态代理这种实现方案更像是AOP编程设计。它倾向并非横向于某种对象而是垂直于某种业务角度。举个例子：小明要削一个苹果给小红吃。我们可以苹果为对象进行削这个方法，然后就能得到一个削好的苹果。而AOP却是你不需要知道苹果这个对象，你可以直接把苹果传递给小红，但是在传递的过程中，我们可以针对这个传递的方法来判断我们当前的业务是否需要进行削，如果要则削皮，不要则不削。同样，即使你传递的是梨，我们也可以通过同样的方式来确定是否要削。这就像埋点，我们需要在onClick()的方法实现中记录埋点，但是我们也可以通过记录onClick()方法触发的时候进行埋点记录，这样就不用在每个onClick()的方法中实现埋点了。

好了题外话不多说，我们进入正题。对于引入视图代理和一级缓存在MVP中，首先Presenter是不改变的，它和最初的样子是一样的依然都是持有一个mView对象，但是持有的并不是真正的View。而现在的mView呢也依然是最初的mView，不做任何改变。唯一改变的就是我们新引入了一个代理的对象，现在我们来看看这个代理对象是如何实现的

```java
public abstract class AbstractViewCacheProxy<T extends IView> implements InvocationHandler {

    /* 如果是weakhashmap。Fragment destroy view就会回收数据了 */
    private final Map<Method, Object[]> mViewCaches = new HashMap<>();
    private WeakReference<T> mView;

    public T proxy(Class<T> viewClass) {
        if (viewClass == null) {
            throw new NullPointerException("Proxy class is NULL, vmProxy is NULL!");
        }
        return (T) Proxy.newProxyInstance(viewClass.getClassLoader(), new Class[]{viewClass}, this);
    }

    void bind(T view) {
        if (view == null) {
            return;
        }
        unBind();
        mView = new WeakReference<>(view);

        for (Method method : mViewCaches.keySet()) {
            invokeMethod(view, method, mViewCaches.get(method));
            LogHelperUtil.i("AbstractViewCacheProxy-bind: ", method.getName());
        }

        view.bindProxyFinish();
    }

    void unBind() {
        if (mView != null) {
            mView.clear();
            mView = null;
        }
    }

    boolean isBind() {
        return mView != null && mView.get() != null;
    }

    void destroy() {
        unBind();
        mViewCaches.clear();
        onDestroy();
    }

    /* 请在此处释放和清理资源 */
    protected abstract void onDestroy();

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (isCacheMethod(method)) {
            mViewCaches.put(method, args);
        }

        if (mView != null && mView.get() != null) {
            return invokeMethod(mView.get(), method, args);
        }
        return null;
    }

    private boolean isCacheMethod(Method method) {
        CacheMethod cacheMethod = method.getAnnotation(CacheMethod.class);
        return cacheMethod != null && cacheMethod.isCached();
    }

    private Object invokeMethod(Object view, Method method, Object[] args) {
        if (view == null || method == null) {
            return null;
        }
        try {
            return method.invoke(view, args);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```



由以上代码可以看出，这是一个InvocationHandler对象，这个对象主要是在生成代理的时候需要传入，在调用代理对象方法的时候会调用该对象子类中的invoke(Object proxy, Method method, Object[] args);方法，而我们便可以在该方法中做我们想做的事。

本类中的字段说明：

```java
private final Map<Method, Object[]> mViewCaches = new HashMap<>();
private WeakReference<T> mView;
```

这两个字段中：

- mViewCaches用来存储调用的方法和调用的参数

- mView则是真实的View对象，是一个弱引用对象。

本类中的一些重点方法说明：

- 首先是public T proxy(Class<T> viewClass)方法，该方法主要是通过Proxy.newProxyInstance()构建一个动态代理的对象，而这个代理对象的InvocationHandler就是本类，而我们所代理的class就是我们的IContract.View.class。
- void bind(T view)和void unBind()方法，其目的和之前所提到VM中的bind()和unbind()是一样的，就是为了将真实的View提供到代理对象中，只是这里是代理对象而之前是一个VM对象。但是在这里的bind()方法不只是简单的绑定一个View，它还做了一个事情就是将之前调用过的方法和参数通过method.invoke(view, args)的方法传递给最新绑定的View，这样是不是就相当于我们跑了一遍Presenter中请求的数据，然后将数据返回给View中呢？而不同的是，我们所提供给View中的数据是缓存且最新的缓存。
- public Object invoke(Object proxy, Method method, Object[] args)这个方法主要是在当我们的代理对象的任一方法被调用的时候，则会回调此方法。

- private boolean isCacheMethod(Method method)这个方法是用于获取当前代理对象调用的方法是否需要被缓存，其理论是通过获取动态注解来判断是否需要被缓存。

看到这里也许聪明的你可能大致就已经明白了我的意图。所谓视图代理对象的实现其实主要是通过动态代理的方式生成一个代理类，当视图代理的方法被主动调用的时候我们通过运行时注解来判断此方法是否需要被记录，如果要则存入mViewCaches后调用，如果不要则直接调用该方法即可。不论View是否有效，我们的视图代理对象方法都会被成功的执行，但是会不会具体的落实到真实的View中就不一定了。即使这次不会落实在View中也不怕，在View下一次绑定我们的视图代理对象的时候，我们依然会在bind()方法中回调之前的调用记录，将最新的缓存数据传递给View，这样做既可以保证数据更新的及时，也可以保证每一次请求的有效性且有价值，因为它基本一定会被应用到View中，而不会因为View的离开而导致本次请求的数据浪费了。

接下来我们再来看看View中是如何与ViewProxy(视图代理)进行链接的

```java
public class FontDetailView extends AbstractFragmentView<IFontDetail.Presenter, FontDetailViewProxy>
        implements IFontDetail.View {

    private FontDetailLoopPagerAdapter mAdapter;
    private AlertDialog mInDataDialog = null;
    private ViewHolder mViewHolder;

    @Override
    protected FontDetailViewProxy onCreateViewProxy() {
        return new FontDetailViewProxy();
    }

    @Override
    protected FontDetailPresenter onCreatePresenter(FontDetailViewProxy viewProxy) {
        return new FontDetailPresenter(viewProxy.proxy(IFontDetail.View.class));
    }
    ...
}
```

这是项目C中FontDetailView.java的部分代码  
上面的代码是继承Fragment的，因为它需要活在ViewPager中。在这里，我贴出  
的是具体实现的View而不是抽象的View，其目的是为了能够更好的解析框架的实现与使用。

我们可以很直观的看到两个方法：onCreateViewProxy()和onCreatePresenter(FontDetailViewProxy viewProxy)，这两个方法会在Fragment的构造方法中调用。在FontDetailView的super的构造方法中我们先是调用了onCreateViewProxy()方法来创建一个具体的FontDetailViewProxy对象，该视图代理对象是AbstractViewCacheProxy的实现子类，接着会将创建好的ViewProxy对象作为参数传递给onCreatePresenter(ViewProxy)中，然后在onCreatePresenter(ViewProxy)方法中我们构建了一个Presenter，并且我们通过ViewProxy.proxy(IContract.View.class)的方式创建了一个代理对象传递给了Presenter，而这个代理对象的InvocationHandler就是我们所创建的FontDetailViewProxy对象了。

经过这两个方法的调用，View和Proxy之间的对象就构建完成了，在抽象的View中会在onActivityCreated()的时候自动将自身绑定到ViewProxy中，具体的代码我就描述了就是调用了bind()方法罢了。

好了，在View中创建了ViewProxy并且将ViewProxy传递给Presenter接着将View和ViewProxy两者进行绑定的步骤以及实现我们已经大致了解了，接下来我们将要讨论如何对制定的调用方法及其参数进行缓存了。

讨论调用方法的缓存其实就是在讨论ViewProxy中的isCacheMethod(Method)方法了，让我们再一次回顾该方法的代码实现：

```java
private boolean isCacheMethod(Method method) {
    CacheMethod cacheMethod = method.getAnnotation(CacheMethod.class);
    return cacheMethod != null && cacheMethod.isCached();
}
```

代码很简单，就是从method方法对象中获取指定的CacheMethod注解，如果获取到了并且isCached()方法为true则返回真，否则返回假。那么也就是说，其实我们只需要在需要缓存的方法前加上@CacheMethod这个注解，则该方法对象以及方法的调用参数就会被缓存了。接下来我们一起来看看CacheMethod这个注解的代码：

```java
public  @interface CacheMethod {  
boolean isCached() default true;  
}

```

注解的代码非常的简单，就是单纯的一个isCached()方法且默认值为true。我们再来看看该注解的使用：

IFontDetail.java

```java
public interface IFontDetail {
   interface View {
       ...
       @CacheMethod
       void updateBtnText(int resId);
       ...
   }
   interface Presenter {
       ...
   }
}
```

由于不相关的接口过多，这里进行了省略只显示出了需要分析的方法

由代码中我们可以看出，使用注解的方式非常的简单，只需要在你想要缓存的方法前加上这个注解即可，接着MVP中的Proxy则会自动将该方法进行缓存和记录，等到再次bind()的时候则会回调此方法。

至此，MVP+Proxy+Cache的介绍基本可以落下帷幕了，但是依然有一些细节是需要注意的：

- bind()方法会被调用几次？如果只是一次那缓存还有何用？

- bind()方法只会在onActivityCreated()方法中被调用，也就是说onActivityCreated()方法被执行了几次bind()方法就会被调用几次，而在onDestroyView()的时候会调用unbind()方法进行接触绑定。

- 假设 1.在一个Activity中只有一个Fragment，如果一直保持在前台那么bind()只会被调用一次，这个方法缓存意义不大，但是如果当前Activity被放在后台了系统调用了当前Fragment的onDestroyView()方法但是没有调用onDestroy()方法，那么当这个Fragment再次显示的时候方法缓存的功效就很明显了，它会在最快的时间内恢复视图在销毁前的状态。

- 假设 2.在一个Activity中有多个Fragment并用ViewPager来组合，比如项目C的首页则是4个Fragment的组合了。此时ViewPager的adapter是一个FragmentPagerAdapter，在这个Adapter的内部会缓存Fragment。当我们将ViewPager从第一页滑动到第三页的时候，此时第一页的onDestroyView()方法则会被调用，当我们滑动到第二页或者第一页的时候则会调用Fragment的onCreateView()-onActivityCreated()此时利用方法缓存的形式就可以在最快速度且不需要做任何网络以及数据库请求的情况下恢复Fragment在销毁前的状态。

CacheMethod能活多久？它在何时清除？

方法的缓存是在Presenter首次调用指定的缓存方法的时候开始进行缓存的，而缓存的数据就会在Activity.onDestroy()时被全部清空。代理对象的生命周期和View的生命周期是一样的，当View被彻底的Destroy掉后，代理对象也会跟着一起销亡。

可能还会有更多的问题是值得探讨的，但本节的内容也差不多结束了，最后我们再用时序图的方式来对本节内容的执行过程做一个演示以及章节总结：

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt5NO8hcr6DdneckEjQJicH4FOeK8LDhVwQsbvK7OeZGKhj9EEkUlxP5u1ibpx6eY3jNIFSShick8z1bA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

上面的时序图只是简单的描述了一下彼此之间的通信过程，其中ViewProxy和Proxy可以看做是同一个对象，只是被分解为了两个部分。

总结：在MVP中通过动态代理的模式将Presenter和View之间进行解耦，解决了很多之前使用VM时所引发的问题，比如在面对复杂视图的时候我们该如何解析VM呢？而在Proxy的方式中，我们无需考虑此问题，因为Proxy是面向调用方法进行缓存而并非对象缓存，被调用者发生了什么事件Proxy则记录什么事件，等被调用者回来后Proxy再还给他就是了。对于局部刷新也是如此。对于显示Toast和dialog等方法，我们也可以正常的使用，对于这类方法我们无需缓存，因为只有当View活着的时候，我们才有必要去显示这些东西，当View离开后一般情况下是不需要显示这类视图的，如果有特定的需求那可做特别的处理。

最后我们再来探讨一下它的优缺点：

- **优点**

- 对于一个Activity对应多个Fragment的情况下使用代理和缓存模式是非常可靠和有效的，并且保证了每一次请求的数据的有效性，而不是当View一解除就deprecate掉。

- 对于方法的缓存可控性高，有需要则缓存，无需要则不缓存。

- 在不影响不改变Presenter的情况下，解除Presenter与View之间的循环引用，完美解决Context内存泄露。

- 架构逻辑清晰，项目交接容易

- **缺点**

- 架构逻辑需要发费一定的学习成本。

- 使用简单但是不适合一次性的功能Demo。



### MVC和MVP的比较

经过了前面几节的内容讲解，我们大概能明白MVP和MVC架构的大致实现和思想方式了。而面对这两种架构方案，我们一起从几个维度来对他们做一个简单的比较。

从上来看：

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt5NO8hcr6DdneckEjQJicH4FIIGcibaDh98c1yZjXIEw82AVNXTQjZCvnhcoIzpTwNRLph4lBNUQQ4w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



MVC和MVP之间还有很多的不同在这里我们就不再一一列举了，在实践中会有跟多的体会。

由以上的表格我们可以很明显的看出两者之间的差异，各有各的优势。至于在实际的开发中到底要使用MVC还是MVP这该由开发者自己定义了。

不知不觉已经到了本文的尾声，接下来我们来对本文做一个简单的回顾：

- 在文章的开始我们讨论了在Android中传统的开发模式MVC，并且对其进行了基本的介绍，并通过部分的代码进行事例的讲解，最后以时序图的方式结束了本节的内容。

- 接着我们讨论了MVP的基础内容，同样也给出了部分code的方式来进行实例的讲解，最后也给出了层次间的时序图，并且提出了一些优缺点。

- 在来就是讲解MVP分别在曾经的项目A、项目B和项目C中的使用已经MVP中的阶梯式的扩展与进阶型的讨论，分别以基础-视图-视图代理的方式来对MVP进行不同的改造，于此同时我们也给出了不少的code碎片进行更深入的讲解。

- 最后就是已表格的方式对MVC和MVP之间进行一个非常简单的比较。

虽然MVP是一款非常优秀的架构，但是再优秀的架构也还是会有缺陷的。在技术的世界中，没有最完美的架构，只有最符合需求的架构，尽管再优秀在完美的架构也还是会有很多不足之处的。在MVP中也是存在不少的不足之处，例如：在构建View和Presenter的时候，我们需要多写大量的冗余接口，这无非是增加了额外的代码量。还有就是假设我需要新增方法或者修改某个方法的参数、返回值等，则至少需要变动3个以上的文件，View，Presenter，以及IContract接口。这些都是MVP的不足之处。但是往往我们不能因为某些可以容忍的不足而放弃，也许放弃可以加快眼前的步伐，但对于未来将深陷到难以自拔了。





### 总结

在当下Android开发的技术中已经有了非常多的Android开发架构方案，除了MVC、MVP以外还有MVVM、Clean、Flux以及Google最新推出的Lifecycle+ViewMode+Repository的架构方案。无论是那种架构，他们都拥有自己的特点，适用于不同的需求定制，各有各的好与坏。

在MVP的架构方案上，网络上也有非常多的变种与发展，我提出的MVP+Proxy+Cache的方案在目前看来尚未在项目中产生问题，但是我深知这是远远不够的，例如：在Cache的时候其实我们是否应该加入一些过期的机制，或者是否何时需要再次强制拉取更新数据等等，不少的疑问还需要做更多的探索与发现，而这些探索将会伴随着未来更多变化的需求来进行改变和提升的。

在未来的日子中，我们将会更加努力的提升当下的架构水平与开发能力，在现有的基础上做出更大的突破与进步，使得开发和迭代的速度更加迅速，并且尽可能的努力让眼前的项目保持一个更加强有力的健壮性，这也一直都会是我们不断追逐的目标。

最后还是那句话：世界上没有什么最好的架构，只有最符合需求的架构。






