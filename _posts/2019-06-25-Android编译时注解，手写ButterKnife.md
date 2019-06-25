---
layout:     post
title:      "Android编译时注解，手写ButterKnife"
subtitle:   ""
date:       2019-06-25 15:55:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### apt介绍

作为Android程序员应该绝大部分分人都用过ButterKnife，Retrofit等框架，这些框架只需要在用的时候使用注解，就可以直接使用了，非常方便。并且这些框架并没有减少性能。

那么这些框架做了哪些东西呢？

我们以ButterKnife为例：

```java
@BindView(R.id.textview)
TextView textview;

private Unbinder bind;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    bind = ButterKnife.bind(this);

    textview.setText("dfjslafjsalfls");
}

@Override
protected void onDestroy() {
    bind.unbind();
    super.onDestroy();
}
```

上面是我们平常使用的时候所使用的代码，这些代码到底做了什么呢？

1. 我们给元素写上注解
2. 在编译器执行的环节，系统收集我们所写注解的元素，并将这些元素组合成Java代码`MainActivity_ViewBinding`类
3. 当我们调用`ButterKnife.bind`的时候，通过动态注入的方式，将`MainActivity`和`MainActivity_ViewBinding`关联起来，并给所有的注解所在的元素赋值。

而这些东西的核心就是在编译期间生成我们所需要的Java文件，这样我们在使用的时候就基本没有性能的影响。

网上找到的APT的流程图：



![](https://upload-images.jianshu.io/upload_images/1599843-297cf01353df93cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)





### 手写ButterKnife

#### 通过上面的图我们先生成这些module

![](https://upload-images.jianshu.io/upload_images/1479785-5cd7b549771c73ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/404/format/webp)

app - 主项目  
butterknife - android lib  
annotaion - java lib  
compiler - java lib



#### 在这些module的配置文件中配置

1. **project的build.gradle**

```java
// 如果你的gradle版本是3.0以上则不需要配置
buildscript {

    repositories {
        google()
        jcenter()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.0.0'

        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        mavenCentral()
    }
}
```



**2.compiler的build.gradle**

添加auto-service和javapoet这两个lib是用来方便我们生成Java代码的。

auto-service： [https://github.com/google/auto](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fgoogle%2Fauto)  
javapoet： [http://blog.csdn.net/crazy1235/article/details/51876192](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fcrazy1235%2Farticle%2Fdetails%2F51876192)



```java
dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
    compile 'com.google.auto.service:auto-service:1.0-rc3'
    compile 'com.squareup:javapoet:1.8.0'
    implementation project(':annotation')
}
```

**3.app的build.gradle**

```java
// gradle版本小于3.0
apply plugin: 'com.neenbedankt.android-apt'

compile project(':butterknife')
compile project(':annotation')
apt project(':compiler')

// gradle版本大于3.0
compile project(':butterknife')
compile project(':annotation')
annotationProcessor project(':compiler')
```



#### 在annotation中写上注解

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.CLASS)
public @interface BindView {
    int value();
}
```



#### 在compiler中编写注解生成器

java文档: [http://www.yq1012.com/api/index.html-overview-summary.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.yq1012.com%2Fapi%2Findex.html-overview-summary.html)  
javax.annotation.processing包  
javax.lang.model.element包

1. 创建类processor继承AbstractProcessor

```java
@AutoService(Processor.class)
public class ButterKnifeProcessor extends AbstractProcessor {
    private Elements elementUtils;
    private Filer filer;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        elementUtils = processingEnvironment.getElementUtils();
        filer = processingEnvironment.getFiler();
    }

    // 指定SourceVersion
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }        
}
```

指定我们所需要的annotation

```java
// 指定processortype
public Set<String> getSupportedAnnotationTypes() {
    Set<String> types = new LinkedHashSet<>();
    Set<Class<? extends Annotation>> supportedAnnotations = getSupportedAnnotations();
    for (Class<? extends Annotation> supportedAnnotation : supportedAnnotations) {
        types.add(supportedAnnotation.getCanonicalName());
    }
    return types;
}

private Set<Class<? extends Annotation>> getSupportedAnnotations() {
    Set<Class<? extends Annotation>> annotations = new LinkedHashSet<>();
    annotations.add(BindView.class);
    return annotations;
}
```

3.在process中处理annotation

首先将所有获取到的BindView细分到每个Activity中

```java
Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(BindView.class);

// 将获取到的bindview细分到每个class
Map<Element, List<Element>> elementMap = new LinkedHashMap<>();

for (Element element : elements) {
    // 返回activity
    Element enclosingElement = element.getEnclosingElement();

    List<Element> bindViewElements = elementMap.get(enclosingElement);
    if (bindViewElements == null) {
        bindViewElements = new ArrayList<>();
        elementMap.put(enclosingElement, bindViewElements);
    }
    bindViewElements.add(element);
}
```

通过遍历的方式处理每个细分的Activity

```java
// 生成代码
for (Map.Entry<Element, List<Element>> entrySet : elementMap.entrySet()) {
    Element enclosingElement = entrySet.getKey();
    List<Element> bindViewElements = entrySet.getValue();

    // public final class xxxActivity_ViewBinding implements Unbinder
    // 获取activity的类名
    String activityClassNameStr = enclosingElement.getSimpleName().toString();
    System.out.println("------------->" + activityClassNameStr);
    ClassName activityClassName = ClassName.bestGuess(activityClassNameStr);
    ClassName unBinderClassName = ClassName.get("com.fastaoe.butterknife", "Unbinder");
    TypeSpec.Builder classBuilder =
            TypeSpec.classBuilder(activityClassNameStr + "_ViewBinding")
                    .addModifiers(Modifier.FINAL, Modifier.PUBLIC)
                    .addSuperinterface(unBinderClassName)
                    // 添加属性 private MainActivity target;
                    .addField(activityClassName, "target", Modifier.PRIVATE);

    // unbind()
    ClassName callSuperClassName = ClassName.get("android.support.annotation", "CallSuper");
    MethodSpec.Builder unbindMethodBuilder = MethodSpec.methodBuilder("unbind")
            .addAnnotation(Override.class)
            .addAnnotation(callSuperClassName)
            .addModifiers(Modifier.PUBLIC, Modifier.FINAL);

    // 构造函数
    MethodSpec.Builder constructorMethodBuilder = MethodSpec.constructorBuilder()
            .addParameter(activityClassName, "target")
            .addModifiers(Modifier.PUBLIC)
            // this.target = target
            .addStatement("this.target = target");

    for (Element bindViewElement : bindViewElements) {
        // textview
        String fieldName = bindViewElement.getSimpleName().toString();
        // Utils
        ClassName utilsClassName = ClassName.get("com.fastaoe.butterknife", "Utils");
        // R.id.textview
        int resourceId = bindViewElement.getAnnotation(BindView.class).value();
        // target.textview = Utils.findViewById(target, R.id.textview)
        constructorMethodBuilder.addStatement("target.$L = $T.findViewById(target, $L)", fieldName, utilsClassName, resourceId);
        // target.textview = null
        unbindMethodBuilder.addStatement("target.$L = null", fieldName);
    }


    classBuilder.addMethod(unbindMethodBuilder.build())
            .addMethod(constructorMethodBuilder.build());

    // 获取包名
    String packageName = elementUtils.getPackageOf(enclosingElement).getQualifiedName().toString();

    try {
        JavaFile.builder(packageName, classBuilder.build())
                .addFileComment("自己写的ButterKnife生成的代码，不要修改！！！")
                .build().writeTo(filer);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

#### 在butterknife中编写注入代码

```java
public class ButterKnife {

    public static Unbinder bind(Activity activity) {
        try {
            Class<? extends Unbinder> bindClazz = (Class<? extends Unbinder>)
                    Class.forName(activity.getClass().getName() + "_ViewBinding");
            // 构造函数
            Constructor<? extends Unbinder> bindConstructor = bindClazz.getDeclaredConstructor(activity.getClass());

            Unbinder unbinder = bindConstructor.newInstance(activity);
            return unbinder;
        } catch (Exception e) {
            e.printStackTrace();
        }

        return Unbinder.EMPTY;
    }
}
```

#### 使用

我们的使用方式就和正真的ButterKnife一样了：

```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.textview)
    TextView textview;

    private Unbinder bind;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bind = ButterKnife.bind(this);

        textview.setText("修改后的文字");
    }

    @Override
    protected void onDestroy() {
        bind.unbind();
        super.onDestroy();
    }
}
```

### 最后

[附上代码](https://link.jianshu.com/?t=https%3A%2F%2Fgithub.com%2Fjsntjinjin%2FButterKnifeDemo.git)




