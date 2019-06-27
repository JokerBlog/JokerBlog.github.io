---
layout:     post
title:      "简单工厂模式（SimpleFactoryPattern）"
subtitle:   ""
date:       2019-06-27 11:13:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### 1. 含义

- 简单工厂模式又叫静态方法模式（因为工厂类定义了一个静态方法）
- 现实生活中，工厂是负责生产产品的；同样在设计模式中，简单工厂模式我们可以理解为负责生产对象的一个类，称为“工厂类”。

  

### 2. 解决的问题

将“类实例化的操作”与“使用对象的操作”分开，让使用者不用知道具体参数就可以实例化出所需要的“产品”类，从而避免了在客户端代码中显式指定，实现了解耦。

> 即使用者可直接消费产品而不需要知道其生产的细节





### 3.模式原理

#### 3.1 模式组成

抽象产品（Product）—— 具体产品的父类    描述产品的公共接口
具体产品（Concrete Product）——抽象产品的子类；工厂类创建的目标类    描述生产的具体产品
工厂（Creator）——被外界调用  根据传入不同参数从而创建不同具体产品类的实例

---

#### 3.2 使用步骤

- 创建**抽象产品类**& 定义具体产品的公共接口；
- 创建**具体产品类**（继承抽象产品类） & 定义生产的具体产品；
- 创建**工厂类**，通过创建静态方法根据传入不同参数从而创建不同具体产品类的实例；
- 外界通过调用工厂类的静态方法，**传入不同参数**从而创建不同**具体产品类的实例**

### 4.实例

接下来我用一个实例来对简单工厂模式进行更深一步的介绍。

#### 4.1 实例概况

背景：小成有一个塑料生产厂，用来做塑料加工生意
目的：最近推出了3个产品，小成希望使用简单工厂模式实现3款产品的生产

#### 4.2 使用步骤

实现代码如下：

###### 步骤1. 创建抽象产品类，定义具体产品的公共接口

```java
abstract class Product{
    public abstract void Show();
}
```

###### 步骤2.创建具体产品类（继承抽象产品类），定义生产的具体产品

```java
//具体产品类A
class  ProductA extends  Product{

    @Override
    public void Show() {
        System.out.println("生产出了产品A");
    }
}

//具体产品类B
class  ProductB extends  Product{

    @Override
    public void Show() {
        System.out.println("生产出了产品C");
    }
}

//具体产品类C
class  ProductC extends  Product{

    @Override
    public void Show() {
        System.out.println("生产出了产品C");
    }
}

```

##### 步骤3.创建工厂类，通过创建静态方法从而根据传入不同参数创建不同具体产品类的实例

```java
class  Factory {
    public static Product Manufacture(String ProductName){
//工厂类里用switch语句控制生产哪种商品；
//使用者只需要调用工厂类的静态方法就可以实现产品类的实例化。
        switch (ProductName){
            case "A":
                return new ProductA();

            case "B":
                return new ProductB();

            case "C":
                return new ProductC();

            default:
                return null;

        }
    }
}

```

**步骤4.**外界通过调用工厂类的静态方法，传入不同参数从而创建不同具体产品类的实例

```java
//工厂产品生产流程
public class SimpleFactoryPattern {
    public static void main(String[] args){
        Factory mFactory = new Factory();

        //客户要产品A
        try {
//调用工厂类的静态方法 & 传入不同参数从而创建产品实例
            mFactory.Manufacture("A").Show();
        }catch (NullPointerException e){
            System.out.println("没有这一类产品");
        }

        //客户要产品B
        try {
            mFactory.Manufacture("B").Show();
        }catch (NullPointerException e){
            System.out.println("没有这一类产品");
        }

        //客户要产品C
        try {
            mFactory.Manufacture("C").Show();
        }catch (NullPointerException e){
            System.out.println("没有这一类产品");
        }

        //客户要产品D
        try {
            mFactory.Manufacture("D").Show();
        }catch (NullPointerException e){
            System.out.println("没有这一类产品");
        }
    }
}

```

### 6.缺点

工厂类集中了所有实例（产品）的创建逻辑，一旦这个工厂不能正常工作，整个系统都会受到影响；
违背“开放 - 关闭原则”，一旦添加新产品就不得不修改工厂类的逻辑，这样就会造成工厂逻辑过于复杂。
简单工厂模式由于使用了静态工厂方法，静态方法不能被继承和重写，会造成工厂角色无法形成基于继承的等级结构。



### 7.应用场景

在了解了优缺点后，我们知道了简单工厂模式的应用场景：

客户如果只知道传入工厂类的参数，对于如何创建对象的逻辑不关心时；

 当工厂类负责创建的对象（具体产品）比较少时。



#### 案例一：

1.工厂模式也是我们最常见的一种模式了，可以用来创建多个不同的实例对象。Android代码中最常见的应该是对Fragment的集中管理了。用Fragment工厂，创建出不同的fragment。

 2.eg: 现在的app大多数都是由少数几个activity和众多的fragment组成，那么针对这些fragment，我们可以开辟一个工厂，针对不同的需求生产不同的fragment，请参考如下代码：

```java
public class FragmentFactory {

    //将已经new 出来的fragment储存起来
    private static HashMap<Integer, BaseFragment> fragmentMap = new HashMap<Integer, BaseFragment>();

    public static BaseFragment createFragment(int position){
        //从集合中取，没有的话再newfragment
        BaseFragment fragment = fragmentMap.get(position);
        if(fragment==null){
            switch (position) {
            case 0:
                fragment = new HomeFragment();
                break;
            case 1:
                fragment = new AppFragment();
                break;
            case 2:
                fragment = new GameFragment();
                break;
            case 3:
                fragment = new SubjectFragment();
                break;
            case 4:
                fragment = new RecommendFragment();
                break;
            case 5:
                fragment = new CategoryFragment();
                break;
            case 6:
                fragment = new HotFragment();
                break;

            default:
                break;
            }
            fragmentMap.put(position, fragment);
        }

        return fragment;
    }
}

```



#### 案例二: 工厂模式优化

##### 背景:采用简单工厂模式

##### 冲突:

##### 1. 操作成本高：每增加一个接口的子类，必须修改工厂类的逻辑

##### 2. 系统复杂性提高：每增加一个接口的子类，都必须向工厂类添加逻辑

**实例演示**

**步骤1. 创建抽象产品类的公共接口**

Product.java

```java
abstract class Product{
    public abstract void show();
}

```

**步骤2. 创建具体产品类（继承抽象产品类），定义生产的具体产品**

```java
<-- 具体产品类A：ProductA.java -->
public class  ProductA extends  Product{

    @Override
    public void show() {
        System.out.println("生产出了产品A");
    }
}

<-- 具体产品类B：ProductB.java -->
public class  ProductB extends  Product{

    @Override
    public void show() {
        System.out.println("生产出了产品B");
    }
}
```

**步骤3. 创建工厂类**

Factory.java

```java
public class Factory {

    // 定义方法：通过反射动态创建产品类实例
    public static Product getInstance(String ClassName) {

        Product concreteProduct = null;

        try {

            // 1. 根据 传入的产品类名 获取 产品类类型的Class对象
            Class product_Class = Class.forName(ClassName);
            // 2. 通过Class对象动态创建该产品类的实例
            concreteProduct = (Product) product_Class.newInstance();

        } catch (Exception e) {
            e.printStackTrace();
        }

        // 3. 返回该产品类实例
        return concreteProduct;
    }

}
```

**步骤4：外界通过调用工厂类的静态方法（反射原理），传入不同参数从而创建不同具体产品类的实例**

TestReflect.java

```java
public class TestReflect {
    public static void main(String[] args) throws Exception {

       // 1. 通过调用工厂类的静态方法（反射原理），从而动态创建产品类实例
        // 需传入完整的类名 & 包名
        Product concreteProduct = Factory.getInstance("scut.carson_ho.reflection_factory.ProductA");

        // 2. 调用该产品类对象的方法，从而生产产品
        concreteProduct.show();
    }
}
```

如此一来，通过采用反射机制（通过 传入子类名称 & 动态创建子类实例），从而使得在增加产品接口子类的情况下，也不需要修改工厂类的逻辑 & 增加系统复杂度。



#### 案例三：**应用了反射机制的工厂模式再次优化**

**背景**

在上述方案中，通过调用工厂类的静态方法（反射原理），从而动态创建产品类实例（该过程中：需传入完整的类名 & 包名）

**冲突**

开发者 无法提前预知 接口中的子类类型 & 完整类名

**解决方案** 

通过 属性文件的形式（ Properties） 配置所要的子类信息，在使用时直接读取属性配置文件从而获取子类信息（完整类名）

**具体实现**

**步骤1：创建抽象产品类的公共接口**

Product.java

```java
abstract class Product{
    public abstract void show();
}
```

**步骤2. 创建具体产品类（继承抽象产品类），定义生产的具体产品**

```java
<-- 具体产品类A：ProductA.java -->  
publicclass  ProductA extends  Product{  

@Override  
public void show() {  
        System.out.println("生产出了产品A");  
    }  
}  

<-- 具体产品类B：ProductB.java -->  
publicclass  ProductB extends  Product{  

@Override  
public void show() {  
        System.out.println("生产出了产品B");  
    }  
}

```

**步骤3. 创建工厂类**

Factory.java

```java
public class Factory {

    // 定义方法：通过反射动态创建产品类实例
    public static Product getInstance(String ClassName) {

        Product concreteProduct = null;

        try {

            // 1. 根据 传入的产品类名 获取 产品类类型的Class对象
            Class product_Class = Class.forName(ClassName);
            // 2. 通过Class对象动态创建该产品类的实例
            concreteProduct = (Product) product_Class.newInstance();

        } catch (Exception e) {
            e.printStackTrace();
        }

        // 3. 返回该产品类实例
        return concreteProduct;
    }

}
```

**步骤4：创建属性配置文件**

Product.properties

```properties
// 写入抽象产品接口类的子类信息（完整类名）
ProductA = scut.carson_ho.reflection_factory.ProductA
ProductB = scut.carson_ho.reflection_factory.ProductB
```

**步骤5：将属性配置文件 放到src/main/assets文件夹中**

> 若没assets文件夹，则自行创建



**步骤6：在动态创建产品类对象时，动态读取属性配置文件从而获取子类完整类名**  

TestReflect.java

```java
public class TestReflect {  
public static void main(String[] args) throws Exception {  

// 1. 读取属性配置文件  
        Properties pro = new Properties() ;  
        pro.load(this.getAssets().open("Product.properties"));  

// 2. 获取属性配置文件中的产品类名  
        String Classname = pro.getProperty("ProductA");  

// 3. 动态生成产品类实例  
        Product concreteProduct = Factory.getInstance(Classname);  

// 4. 调用该产品类对象的方法，从而生产产品  
        concreteProduct.show();  

}

```








