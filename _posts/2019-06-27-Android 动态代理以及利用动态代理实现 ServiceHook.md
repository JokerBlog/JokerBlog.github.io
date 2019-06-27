---
layout:     post
title:      "Android 动态代理以及利用动态代理实现 ServiceHook"
subtitle:   ""
date:       2019-06-27 14:16:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### **Java 的动态代理**

首先我们要介绍的就是 Java 动态代理，Java 的动态代理涉及到两个类：InvocationHandler 接口和 Proxy 类，下面我们会着重介绍一下这两个类，并且结合实例来着重分析一下使用的正确姿势等。在这之前简单介绍一下 Java 中 class 文件的生成和加载过程，Java 编译器编译好 Java 文件之后会在磁盘中产生 .class 文件。这种 .class 文件是二进制文件，内容是只有 JVM 虚拟机才能识别的机器码，JVM 虚拟机读取字节码文件，取出二进制数据，加载到内存中，解析 .class 文件内的信息，使用相对应的 ClassLoader 类加载器生成对应的 Class 对象：

![](https://mmbiz.qpic.cn/mmbiz_png/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkxrggE9cW0aJkRNibWpL0qjGMh0ibDdloCUQe93CqfK1HVJquHJf3seib6A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

.class 字节码文件是根据 JVM 虚拟机规范中规定的字节码组织规则生成的

通过上面我们知道 JVM 是通过字节码的二进制信息加载类的，那么我们如果在运行期系统中，遵循 Java 编译系统组织 .class 文件的格式和结构，生成相应的二进制数据，然后再把这个二进制数据转换成对应的类，这样就可以在运行中动态生成一个我们想要的类了：

![](https://mmbiz.qpic.cn/mmbiz_png/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkx9eLbSqUfH4jAdBuJ8abib3ELXUPzDaO71trJnCHjn4ApMibm1NcBWhgA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Java 中有很多的框架可以在运行时根据 JVM 规范动态的生成对应的 .class 二进制字节码，比如 ASM 和 Javassist 等，这里就不详细介绍了，感兴趣的可以去查阅相关的资料。这里我们就以动态代理模式为例来介绍一下我们要用到这两个很重要的类，关于动态代理模式，

Java 中有很多的框架可以在运行时根据 JVM 规范动态的生成对应的 .class 二进制字节码，比如 ASM 和 Javassist 等，这里就不详细介绍了，感兴趣的可以去查阅相关的资料。这里我们就以动态代理模式为例来介绍一下我们要用到这两个很重要的类，关于动态代理模式，

![](https://mmbiz.qpic.cn/mmbiz_png/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkxTW1uFPRunaGc5Y4PtvOF7k7pv1ia7Niax3l0gfQP1J6F3fxviatnAF1Og/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

上面就是静态代理模式的类图，当在代码阶段规定这种代理关系时，ProxySubject 类通过编译器生成 .class 字节码文件，当系统运行之前，这个 .class 文件就已经存在了。动态代理模式的结构和上面的静态代理模式的结构稍微有所不同，它引入了一个 InvocationHandler 接口和 Proxy 类。在静态代理模式中，代理类 ProxySubject 中的方法，都指定地调用到特定 RealSubject 中对应的方法，ProxySubject 所做的事情无非是调用触发 RealSubject 对应的方法；动态代理工作的基本模式就是将自己方法功能的实现交给 InvocationHandler 角色，外界对 Proxy 角色中每一个方法的调用，Proxy 角色都会交给 InvocationHandler 来处理，而 InvocationHandler 则调用 RealSubject 的方法，如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_jpg/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkxWaMlMXg0F8c7TOgLQGImqT6X8ib5M3uybWVQBEKku8iawSicJAeZmFCOg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### InvocationHandler接口和Proxy 类

我们来分析一下动态代理模式中 ProxySubject 的生成步骤：

1. 获取 RealSubject 上的所有接口列表；

2. 确定要生成的代理类的类名，系统默认生成的名字为：com.sun.proxy.$ProxyXXXX ；

3. 根据需要实现的接口信息，在代码中动态创建该 ProxySubject 类的字节码；

4. 将对应的字节码转换为对应的 Class 对象；

5. 创建 InvocationHandler 的实例对象 h，用来处理 Proxy 角色的所有方法调用；

6. 以创建的 h 对象为参数，实例化一个 Proxy 角色对象。



具体的代码为：

**Subject.java**

```java
public interface Subject {
    String operation();
}
```

**RealSubject.java**

```java
public class RealSubject implements Subject{
    @Override
    public String operation() {
        return "operation by subject";
    }
}
```

**ProxySubject.java**

```java
public class ProxySubject implements InvocationHandler{
     protected Subject subject;
    public ProxySubject(Subject subject) {
        this.subject = subject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //do something before
        return method.invoke(subject, args);
    }
}

```

**测试代码**

```java
Subject subject = new RealSubject();
ProxySubject proxy = new ProxySubject(subject);
Subject sub = (Subject) Proxy.newProxyInstance(subject.getClass().getClassLoader(),
        subject.getClass().getInterfaces(), proxy);
sub.operation();
```

以上就是动态代理模式的最简单实现代码，JDK 通过使用 java.lang.reflect.Proxy 包来支持动态代理，我们来看看这个类的表述：

```java
Proxy provides static methods for creating dynamic proxy classes and instances, and it is also the 
superclass of all dynamic proxy classes created by those methods.
```

一般情况下，我们使用下面的

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[]interfaces,InvocationHandler h) throws IllegalArgumentException { 
    // 检查 h 不为空，否则抛异常
    if (h == null) { 
        throw new NullPointerException(); 
    } 

    // 获得与指定类装载器和一组接口相关的代理类类型对象
    Class cl = getProxyClass(loader, interfaces); 

    // 通过反射获取构造函数对象并生成代理类实例
    try { 
        Constructor cons = cl.getConstructor(constructorParams); 
        return (Object) cons.newInstance(new Object[] { h }); 
    } catch (NoSuchMethodException e) { throw new InternalError(e.toString()); 
    } catch (IllegalAccessException e) { throw new InternalError(e.toString()); 
    } catch (InstantiationException e) { throw new InternalError(e.toString()); 
    } catch (InvocationTargetException e) { throw new InternalError(e.toString()); 
    } 
}
```

Proxy 类的 getProxyClass 方法调用了 ProxyGenerator 的 generatorProxyClass 方法去生成动态类：

```java
public static byte[] generateProxyClass(final String name, Class[] interfaces)
```

这个方法我们下面将会介绍到，这里先略过，生成这个动态类的字节码之后，通过反射去生成这个动态类的对象，通过 Proxy 类的这个静态函数生成了一个动态代理对象 sub 之后，调用 sub 代理对象的每一个方法，在代码内部，都是直接调用了 InvocationHandler 的 invoke 方法，而 invoke 方法根据代理类传递给自己的 method 参数来区分是什么方法，我们来看看 InvocationHandler 类的介绍：

```java
InvocationHandler is the interface implemented by the invocation handler of a proxy instance.

Each proxy instance has an associated invocation handler. When a method is invoked on a proxy 
instance, the method invocation is encoded and dispatched to the invoke method of its invocation handler.
```

上面提到的一点需要特别注意的是，如果 Subject 类中定义的方法返回值为 8 种基本数据类型，那么在 ProxySubject 类中必须要返回相应的基本类型包装类，即 int 对应的返回为 Integer 等等，还需要注意的是如果此时返回 null，则会抛出 NullPointerException，除此之外的其他情况下返回值的对象必须要和 Subject 类中定义方法的返回值一致，要不然会抛出 ClassCastException。

### **生成源码分析**

那么通过 Proxy 类的 newProxyInstance 方法动态生成的类是什么样子的呢，我们上面也提到了，JDK 为我们提供了一个方法 ProxyGenerator.generateProxyClass(String proxyName,class[] interfaces) 来产生动态代理类的字节码，这个类位于 sun.misc 包中，是属于特殊的 jar 包，于是问题又来了，Android studio 创建的 android 工程是没法找到 ProxyGenerator 这个类的，这个类在 jre 目录下，就算我把这个类相关的 .jar 包拷贝到工程里面并且在 gradle 里面引用它，虽然最后能够找到这个类，但是编译时又会出现很奇葩的问题，所以，没办法喽，android studio 没办法创建普通的 java 工程，只能自己装一个 intellij idea 或者求助相关的同事了。创建好 java 工程之后，使用下面这段代码就可以将生成的类导出到指定路径下面：

```java
public static void generateClassFile(Class clazz,String proxyName)
{
    //根据类信息和提供的代理类名称，生成字节码  
    byte[] classFile = ProxyGenerator.generateProxyClass(proxyName, clazz.getInterfaces());
    String paths = "D:\\"; // 这里写死路径为 D 盘，可以根据实际需要去修改
    System.out.println(paths);
    FileOutputStream out = null;

    try {
        //保留到硬盘中  
        out = new FileOutputStream(paths+proxyName+".class");
        out.write(classFile);
        out.flush();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            out.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

调用代码的方式为：

```java
generateClassFile(ProxySubject.class, "ProxySubject");
```

最后就会在 D 盘（如果没有修改路径）的根目录下面生成一个 ProxySubject.class 的文件，使用 jd-gui 就可以打开该文件：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class ProxySubject
  extends Proxy
  implements Subject
{
  private static Method m1;
  private static Method m3;
  private static Method m2;
  private static Method m0;

  public ProxySubject(InvocationHandler paramInvocationHandler)
  {
    super(paramInvocationHandler);
  }

  public final boolean equals(Object paramObject)
  {
    try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final String operation()
  {
    try
    {
      return (String)this.h.invoke(this, m3, null);
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final String toString()
  {
    try
    {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final int hashCode()
  {
    try
    {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  static
  {
    try
    {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m3 = Class.forName("Subject").getMethod("operation", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
```

可以观察到这个生成的类继承自 java.lang.reflect.Proxy，实现了 Subject 接口，我们在看看生成动态类的代码：

```java
Subject sub = (Subject) Proxy.newProxyInstance(subject.getClass().getClassLoader(),
        subject.getClass().getInterfaces(), proxy);
```

可见这个动态生成类会实现 `subject.getClass().getInterfaces()` 中的所有接口，并且还有一点是类中所有的方法都是 final 的，而且该类也是 final ，所以该类不可继承，最后就是所有的方法都会调用到 InvocationHandler 对象 h 的 invoke() 方法，这也就是为什么最后会调用到 ProxySubject 类的 invoke() 方法了，画一下它们的简单类图：

![](https://mmbiz.qpic.cn/mmbiz_png/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkxgsIN1Mp3TicPHiacibJ1d32qbwriaHmiak53R6pqyicAibAMHWncQ1zB6jyxw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



从这个类图可以很清楚的看明白，动态生成的类 ProxySubject（同名，所以后面加上了 Dynamic）持有了实现 InvocationHandler 接口的 ProxySubject 类的对象 h，然后调用代理对象的 operation 方法时，就会调用到对象 h 的 invoke 方法中，invoke 方法中根据 method 的名字来区分到底是什么方法，然后通过 method.invoke() 方法来调用具体对象的对应方法。



### Android 中利用动态代理实现 ServiceHook

通过上面对 InvocationHandler 的介绍，我们对这个接口应该有了大体的了解，但是在运行时动态生成的代理类有什么作用呢，其实它的作用就是在调用真正业务之前或者之后插入一些额外的操作：

![](https://mmbiz.qpic.cn/mmbiz_png/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkx1zdzyTfYcD3P3tJ1DupvMely2QIp5Xc5NujNTPNichJKEaWeEa1X9WA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



所以简而言之，代理类的处理逻辑很简单，就是在调用某个方法前及方法后插入一些额外的业务。而我们在 Android 中的实践例子就是在真正调用系统的某个 Service 之前和之后选择性的做一些自己特殊的处理，这种思想在插件化框架上也是很重要的。那么我们具体怎么去实现 hook 系统的 Service ，在真正调用系统 Service 的时候附加上我们需要的业务呢，这就需要介绍 ServiceManager 这个类了。

```java
public final class ServiceManager {
    private static final String TAG = "ServiceManager";

    private static IServiceManager sServiceManager;
    private static HashMap<String, IBinder> sCache = new HashMap<String, IBinder>();

    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }

    /**
     * Returns a reference to a service with the given name.
     * 
     * @param name the name of the service to get
     * @return a reference to the service, or <code>null</code> if the service doesn't exist
     */
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return getIServiceManager().getService(name);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

    public static void addService(String name, IBinder service) {
    ...
    }
    ....
}
```

我们可以看到，getService 方法第一步会去 sCache 这个 map 中根据 Service 的名字获取这个 Service 的 IBinder 对象，如果获取到为空，则会通过 ServiceManagerNative 通过跨进程通信获取这个 Service 的 IBinder 对象，所以我们就以 sCache 这个 map 为切入点，反射该对象，然后修改该对象，由于系统的 android.os.ServiceManager 类是 @hide 的，所以只能使用反射，根据这个初步思路，写下第一步的代码：

```java
Class c_ServiceManager = Class.forName("android.os.ServiceManager");
if (c_ServiceManager == null) {
    return;
}

if (sCacheService == null) {
    try {
        Field sCache = c_ServiceManager.getDeclaredField("sCache");
        sCache.setAccessible(true);
        sCacheService = (Map<String, IBinder>) sCache.get(null);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
sCacheService.remove(serviceName);
sCacheService.put(serviceName, service);
```

反射 sCache 这个变量，移除系统 Service，然后将我们自己改造过的 Service put 进去，这样就能实现当调用 ServiceManager 的 getService(String name) 方法的时候，返回的是我们改造过的 Service 而不是系统的原生 Service。

**第二步**

第一步知道了如何去将改造过后的 Service put 进系统的 ServiceManager 中，第二步就是去生成一个 hook Service 了，怎么去生成呢？这就要用到我们上面介绍到的 InvocationHandler 类，我们先获取原生的 Service ，然后通过 InvocationHandler 去构造一个 hook Service，最后通过第一步的步骤 put 进 sCache 这个变量即可，第二步代码：

```java
public class ServiceHook implements InvocationHandler {
    private static final String TAG = "ServiceHook";

    private IBinder mBase;
    private Class<?> mStub;
    private Class<?> mInterface;
    private InvocationHandler mInvocationHandler;

    public ServiceHook(IBinder mBase, String iInterfaceName, boolean isStub, InvocationHandler InvocationHandler) {
        this.mBase = mBase;
        this.mInvocationHandler = InvocationHandler;

        try {
            this.mInterface = Class.forName(iInterfaceName);
            this.mStub = Class.forName(String.format("%s%s", iInterfaceName, isStub ? "$Stub" : ""));
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }

    @Override public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("queryLocalInterface".equals(method.getName())) {
            return Proxy.newProxyInstance(proxy.getClass().getClassLoader(), new Class[] { mInterface },
                    new HookHandler(mBase, mStub, mInvocationHandler));
        }

        Log.e(TAG, "ERROR!!!!! method:name = " + method.getName());
        return method.invoke(mBase, args);
    }

    private static class HookHandler implements InvocationHandler {
        private Object mBase;
        private InvocationHandler mInvocationHandler;

        public HookHandler(IBinder base, Class<?> stubClass,
                           InvocationHandler InvocationHandler) {
            mInvocationHandler = InvocationHandler;

            try {
                Method asInterface = stubClass.getDeclaredMethod("asInterface", IBinder.class);
                this.mBase = asInterface.invoke(null, base);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        @Override public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (mInvocationHandler != null) {
                return mInvocationHandler.invoke(mBase, method, args);
            }
            return method.invoke(mBase, args);
        }
    }
}
```

这里我们以 ClipboardService 的调用代码为例：

```java
IBinder clipboardService = ServiceManager.getService(Context.CLIPBOARD_SERVICE);
String IClipboard = "android.content.IClipboard";

if (clipboardService != null) {
    IBinder hookClipboardService =
            (IBinder) Proxy.newProxyInstance(clipboardService.getClass().getClassLoader(),
                            clipboardService.getClass().getInterfaces(),
                    new ServiceHook(clipboardService, IClipboard, true, new ClipboardHookHandler()));
    //调用第一步的方法
    ServiceManager.setService(Context.CLIPBOARD_SERVICE, hookClipboardService);
} else {
    Log.e(TAG, "ClipboardService hook failed!");
}

```

分析一下上面的这段代码，分析之前，先要看一下 ClipboardService 的相关类图：

![](https://mmbiz.qpic.cn/mmbiz_png/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkx4ptDO3ia2JvBJ2RKlLEycVeQEPkicVqgSCxnSwDZ8x5kMCK1VZNJaiaOg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



从这张类图我们可以清晰的看见 ClipboardService 的相关继承关系，接下来就是分析代码了：

- 调用代码中，Proxy.newProxyInstance 函数的第二个参数需要注意，由于 ClipboardService 继承自三个接口，所以这里需要把所有的接口传递进去，但是如果将第二个参数变更为 `new Class[] { IBinder.class }` 其实也是没有问题的（感兴趣的可以试一下，第一个参数修改为 IBinder.class.getClassLoader()，第二个参数修改为 new Class[]{IBinder.class}，也是可以的），因为实际使用的时候，我们只是用到了 IBinder 类的 queryLocalInterface 方法，其他的方法都没有使用到，接下来我们就会说明 queryLocalInterface 这个函数的作用；

- Proxy.newProxyInstance 函数的第三个参数为 ServcieHook 对象，所以当把这个 Service set 进 sCache 变量的时候，所有调用 ClipboardService 的操作都将调用到 ServiceHook 中的 invoke 方法中；

- 接着我们来看看 ServiceHook 的 invoke 函数，很简单，当函数为 queryLocalInterface 方法的时候返回一个 HookHandler 的对象，其他的情况直接调用 method.invoke 原生系统的 ClipboardService 功能，为什么只处理 queryLocalInterface 方法呢，这个我在博客：java/android 设计模式学习笔记（9）—代理模式 中分析 AMS 的时候已经提到了，asInterface 方法最终会调用到 queryLocalInterface 方法，queryLocalInterface 方法最后的返回结果会作为 asInterface 的结果而返回给 Service 的调用方，所以 queryLocalInterface 方法的最后返回的对象是会被外部直接调用的，这也解释了为什么调用代码中的第二个参数变更为 `new Class[] { IBinder.class }` 也是没有问题，因为第一次调用到 queryLocalInterface 函数之后，后续的所有调用都到了 HookHandler 对象中，动态生成的对象中只需要有 IBinder 的 queryLocalInterface 方法即可，而不需要 IClipboard 接口的其他方法；

- 接下来是 HookHandler 类，首先我们看看这个类的构造函数，第一个参数为系统的 ClipboardService，第二个参数为

```java
Class.forName(String.format("%s%s", iInterfaceName, isStub ? "$Stub" : ""))//"android.content.IClipboard"
```

这个参数咱们对照上面的类图，这个类为 ClipboardService 的父类，它里面有一个 asInterface 的方法，通过反射 asInterface 方法然后将 IBinder 对象变成 IInterface 对象，为什么要这么做，可以去看看我的博客： java/android 设计模式学习笔记（9）—代理模式 中的最后总结，通过 ServiceManager.getService 方法获取一个 IBinder 对象，但是这个 IBinder 对象不能直接调用，必须要通过 asInterface 方法转成对应的 IInterface 对象才可以使用，所以 mBase 对象其实是一个 IInterface 对象：

![](https://mmbiz.qpic.cn/mmbiz_png/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkxicCUZ9hGibGG6EOAFA75eB8JupyI5icZympwBHq4SPIB7OU8iaHKOiauA7g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



最后也证实了这个结果，为什么是 Proxy 对象这就不用我解释了吧；

- 最后是 HookHandler 的 invoke 方法，这个方法调用到了 ClipboardHookHandler 对象，我们来看看这个类的实现：

```java
public class ClipboardHook {

    private static final String TAG = ClipboardHook.class.getSimpleName();

    public static void hookService(Context context) {
        IBinder clipboardService = ServiceManager.getService(Context.CLIPBOARD_SERVICE);
        String IClipboard = "android.content.IClipboard";

        if (clipboardService != null) {
            IBinder hookClipboardService =
                    (IBinder) Proxy.newProxyInstance(IBinder.class.getClassLoader(),
                            new Class[]{IBinder.class},
                            new ServiceHook(clipboardService, IClipboard, true, new ClipboardHookHandler()));
            ServiceManager.setService(Context.CLIPBOARD_SERVICE, hookClipboardService);
        } else {
            Log.e(TAG, "ClipboardService hook failed!");
        }
    }

    public static class ClipboardHookHandler implements InvocationHandler {

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            String methodName = method.getName();
            int argsLength = args.length;
            //每次从本应用复制的文本，后面都加上分享的出处
            if ("setPrimaryClip".equals(methodName)) {
                if (argsLength >= 2 && args[0] instanceof ClipData) {
                    ClipData data = (ClipData) args[0];
                    String text = data.getItemAt(0).getText().toString();
                    text += "this is shared from ServiceHook-----by Shawn_Dut";
                    args[0] = ClipData.newPlainText(data.getDescription().getLabel(), text);
                }
            }
            return method.invoke(proxy, args);
        }
    }
}
```

所以 ClipboardHookHandler 类的 invoke 方法最终获取到了要 hook 的 Service 的 IInterface 对象（即为 IClipboard.Proxy 对象，最后通过 Binder Driver 调用到了系统的 ClipboardService 中），调用函数的 Method 对象和参数列表对象，获取到了这些之后，不用我说了，就可以尽情的去做一些额外的操作了，我这里是在仿照知乎复制文字时，在后面加上类似的版权声明。

##问题讨论

上面就是 ServiceHook 的详细步骤了，了解它必须要对 InvocationHandler 有详细的了解，并且还要去看一下 AOSP 源码，比如要去 hook ClipboardService ，那么就要去先看看 ClipboardService 的源码，看看这个类中每个函数的名字和作用，参数列表中每个参数的顺序和作用，而且有时候这还远远不够，我们知道，随着 Android 每个版本的更新，这些类可能也会被更新修改甚至删除，很有可能对于新版本来说老的 hook 方法就不管用了，这时候必须要去了解最新的源码，看看更新修改的地方，针对新版本去重新制定 hook 的步骤，这是一点需要慎重对待考虑的地方。

##### **步骤**

![](https://mmbiz.qpic.cn/mmbiz_png/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkxbEFvToic57W4uoVicAoOr77gxgkVrCiaAHQoJvHMkm8QJMGyyfibmyZ5MQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_gif/u5mVFfJNlgVk1dWKM1DQwVibwhqr8cmkxAEhMXSQmLqW3g3sVsDr3ic4BO413t9jjs2XVbzKo9Fala4BQ00vTsIw/0?wx_fmt=gif&wxfrom=5&wx_lazy=1)












