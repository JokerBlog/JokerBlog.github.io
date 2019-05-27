---
layout:     post
title:      "4.Flutter MaterialDesgign "
subtitle:   ""
date:       2019-05-27 16:25:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - Flutter
---



Material Design，中文名：质感设计，是由Google推出的全新设计语言，旨在为手机、平板电脑、台式机和其他平台提供更一致、更广泛的外观和感觉。从2014年开始，Android到衍生的Android Wear、Auto和TV，Material Design贯穿其中，成为勾通不同平台、设备的灵魂，让用户在不同平台上也有连贯的体验。为了维护这种一致，Google不允许第三方修改Android Wear、Auto和TV的界面以及交互。

 Flutter提供了一些控件，帮助你构建遵循质感设计(Material Design)的应用程序。一个质感设计的应用程序从MaterialApp控件开始，它在应用程序根目录下建立许多常用的控件，包括Navigator，通过字符串标识来管理堆叠的界面，也称为routes。Navigator让应用程序各个界面之间平滑地过渡。不一定要使用MaterialApp控件，但它使用起来很方便实用。

```
import 'package:flutter/material.dart';
void main() {
  runApp(new MaterialApp(
    title: 'Flutter教程',
    home: new TutorialHome(),
  ));
}
class TutorialHome extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        leading: new IconButton(
          icon: new Icon(Icons.menu),
          tooltip: '导航菜单',
          onPressed: null,
        ),
        title: new Text('实例标题'),
        actions: <Widget>[
          new IconButton(
            icon: new Icon(Icons.search),
            tooltip: '搜索',
            onPressed: null,
          ),
        ],
      ),
      body: new Center(
        child: new Text('你好，世界！'),
      ),
      floatingActionButton: new FloatingActionButton(
        tooltip: '增加',
        child: new Icon(Icons.add),
        onPressed: null,
      ),
    );
  }
}


```

上面的实例基本上是一个质感设计，比如，应用栏有阴影和标题文本自动继承正确的样式，还添加了一个浮动的操作按钮。

需要注意的是，我们再次将控件作为参数传递给其他控件。Scaffold是实现基本质感设计可视化的布局结构，Scaffold控件使用多种不同的控件作为命名参数，并将每一个布置在Scaffold适当的位置。同样的，AppBar控件让我们传递leading、actions和title控件。



![](https://img-blog.csdn.net/20161101195503559)
































