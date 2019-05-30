---
layout:     post
title:      "Flutter Views "
subtitle:   ""
date:       2019-05-30 09:47:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - Flutter
---

#### 1、 Flutter和Android中的View

在Android中，View是屏幕上显示的所有内容的基础， 按钮、工具栏、输入框等一切都是View。 在Flutter中，View相当于是Widget。然而，与View相比，Widget有一些不同之处。 首先，Widget仅支持一帧，并且在每一帧上，Flutter的框架都会创建一个Widget实例树(译者语：相当于一次性绘制整个界面)。 相比之下，在Android上View绘制结束后，就不会重绘，直到调用invalidate时才会重绘。

与Android的视图层次系统不同（在framework改变视图），而在Flutter中的widget是不可变的，这允许widget变得超级轻量。



#### 2、如何更新widget

在Android中，您可以通过直接对view进行改变来更新视图。然而，在Flutter中Widget是不可变的，不会直接更新，而必须使用Widget的状态。

这是Stateful和Stateless widget的概念的来源。一个Stateless Widget就像它的名字，是一个没有状态信息的widget。

当您所需要的用户界面不依赖于对象配置信息以外的其他任何内容时，StatelessWidgets非常有用。

例如，在Android中，这与将logo图标放在ImageView中很相似。logo在运行时不会更改，因此您可以在Flutter中使用StatelessWidget。

如果您希望通过HTTP动态请求的数据更改用户界面，则必须使用StatefulWidget，并告诉Flutter框架该widget的状态已更新，以便可以更新该widget。

这里要注意的重要一点是无状态和有状态 widget的核心特性是相同的。每一帧它们都会重新构建，不同之处在于StatefulWidget有一个State对象，它可以跨帧存储状态数据并恢复它。

如果你有疑问，那么要记住这个规则：如果一个widget发生了变化（例如用户与它交互），它就是有状态的。但是，如果一个子widget对变化做出反应，而其父widget对变化没有反应，那么包含的父widget仍然可以是无状态的widget。

我们来看看你如何使用一个StatelessWidget。一个常见的StatelessWidget是Text。如果你看看Text Widget的实现，你会发现它是一个StatelessWidget的子类:



```dart
new Text(
  'I like Flutter!',
  style: new TextStyle(fontWeight: FontWeight.bold),
);
```

正如你所看到的，Text 没有与之关联的状态信息，它呈现了构造函数中传递的内容，仅此而已。

但是，如果你想让“I Like Flutter”动态变化，例如点击一个FloatingActionButton？这可以通过将Text包装在StatefulWidget中并在点击按钮时更新它来实现，如：

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(new SampleApp());
}

class SampleApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'Sample App',
      theme: new ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: new SampleAppPage(),
    );
  }
}

class SampleAppPage extends StatefulWidget {
  SampleAppPage({Key key}) : super(key: key);

  @override
  _SampleAppPageState createState() => new _SampleAppPageState();
}

class _SampleAppPageState extends State<SampleAppPage> {
  // Default placeholder text
  String textToShow = "I Like Flutter";

  void _updateText() {
    setState(() {
      // update the text
      textToShow = "Flutter is Awesome!";
    });
  }

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text("Sample App"),
      ),
      body: new Center(child: new Text(textToShow)),
      floatingActionButton: new FloatingActionButton(
        onPressed: _updateText,
        tooltip: 'Update Text',
        child: new Icon(Icons.update),
      ),
    );
  }
}
```

#### 


在Android中，您通过XML编写布局，但在Flutter中，您可以使用widget树来编写布局。

这里是一个例子，展示了如何在屏幕上显示一个简单的Widget并添加一些padding。

```dart
@override
Widget build(BuildContext context) {
  return new Scaffold(
    appBar: new AppBar(
      title: new Text("Sample App"),
    ),
    body: new Center(
      child: new MaterialButton(
        onPressed: () {},
        child: new Text('Hello'),
        padding: new EdgeInsets.only(left: 10.0, right: 10.0),
      ),
    ),
  );
}
```



#### 4. 如何在布局中添加或删除组件

在Android中，您可以从父级控件调用addChild或removeChild以动态添加或删除View。 在Flutter中，因为widget是不可变的，所以没有addChild。相反，您可以传入一个函数，该函数返回一个widget给父项，并通过布尔值控制该widget的创建。

例如，当你点击一个FloatingActionButton时，如何在两个widget之间切换：

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(new SampleApp());
}

class SampleApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'Sample App',
      theme: new ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: new SampleAppPage(),
    );
  }
}

class SampleAppPage extends StatefulWidget {
  SampleAppPage({Key key}) : super(key: key);

  @override
  _SampleAppPageState createState() => new _SampleAppPageState();
}

class _SampleAppPageState extends State<SampleAppPage> {
  // Default value for toggle
  bool toggle = true;
  void _toggle() {
    setState(() {
      toggle = !toggle;
    });
  }

  _getToggleChild() {
    if (toggle) {
      return new Text('Toggle One');
    } else {
      return new MaterialButton(onPressed: () {}, child: new Text('Toggle Two'));
    }
  }

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text("Sample App"),
      ),
      body: new Center(
        child: _getToggleChild(),
      ),
      floatingActionButton: new FloatingActionButton(
        onPressed: _toggle,
        tooltip: 'Update Text',
        child: new Icon(Icons.update),
      ),
    );
  }
}
```



#### 5.在Android中，可以通过View.animate()对视图进行动画处理，那在Flutter中怎样才能对Widget进行处理

在Flutter中，可以通过动画库给widget添加动画。

在Android中，您可以通过XML创建动画或在视图上调用.animate()。在Flutter中，您可以将widget包装到Animation中。

与Android相似，在Flutter中，您有一个AnimationController和一个Interpolator， 它是Animation类的扩展，例如CurvedAnimation。您将控制器和动画传递到AnimationWidget中，并告诉控制器启动动画。

让我们来看看如何编写一个FadeTransition，当您按下时会淡入一个logo:



```dart
import 'package:flutter/material.dart';

void main() {
  runApp(new FadeAppTest());
}

class FadeAppTest extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'Fade Demo',
      theme: new ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: new MyFadeTest(title: 'Fade Demo'),
    );
  }
}

class MyFadeTest extends StatefulWidget {
  MyFadeTest({Key key, this.title}) : super(key: key);
  final String title;
  @override
  _MyFadeTest createState() => new _MyFadeTest();
}

class _MyFadeTest extends State<MyFadeTest> with TickerProviderStateMixin {
  AnimationController controller;
  CurvedAnimation curve;

  @override
  void initState() {
    controller = new AnimationController(duration: const Duration(milliseconds: 2000), vsync: this);
    curve = new CurvedAnimation(parent: controller, curve: Curves.easeIn);
  }

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text(widget.title),
      ),
      body: new Center(
          child: new Container(
              child: new FadeTransition(
                  opacity: curve,
                  child: new FlutterLogo(
                    size: 100.0,
                  )))),
      floatingActionButton: new FloatingActionButton(
        tooltip: 'Fade',
        child: new Icon(Icons.brush),
        onPressed: () {
          controller.forward();
        },
      ),
    );
  }
}
```



#### 6.如何使用Canvas draw/paint

在Android中，您可以使用Canvas在屏幕上绘制自定义形状。

Flutter有两个类可以帮助您绘制画布，CustomPaint和CustomPainter，它们实现您的算法以绘制到画布。

在这个人气较高的的StackOverFlow答案中，您可以看到签名painter是如何实现的。

请参阅[https://stackoverflow.com/questions/46241071/create-signature-area-for-mobile-app-in-dart-flutter](https://stackoverflow.com/questions/46241071/create-signature-area-for-mobile-app-in-dart-flutter)

```dart
import 'package:flutter/material.dart';
class SignaturePainter extends CustomPainter {
  SignaturePainter(this.points);
  final List<Offset> points;
  void paint(Canvas canvas, Size size) {
    var paint = new Paint()
      ..color = Colors.black
      ..strokeCap = StrokeCap.round
      ..strokeWidth = 5.0;
    for (int i = 0; i < points.length - 1; i++) {
      if (points[i] != null && points[i + 1] != null)
        canvas.drawLine(points[i], points[i + 1], paint);
    }
  }
  bool shouldRepaint(SignaturePainter other) => other.points != points;
}
class Signature extends StatefulWidget {
  SignatureState createState() => new SignatureState();
}
class SignatureState extends State<Signature> {
  List<Offset> _points = <Offset>[];
  Widget build(BuildContext context) {
    return new GestureDetector(
      onPanUpdate: (DragUpdateDetails details) {
        setState(() {
          RenderBox referenceBox = context.findRenderObject();
          Offset localPosition =
          referenceBox.globalToLocal(details.globalPosition);
          _points = new List.from(_points)..add(localPosition);
        });
      },
      onPanEnd: (DragEndDetails details) => _points.add(null),
      child: new CustomPaint(painter: new SignaturePainter(_points)),
    );
  }
}
class DemoApp extends StatelessWidget {
  Widget build(BuildContext context) => new Scaffold(body: new Signature());
}
void main() => runApp(new MaterialApp(home: new DemoApp()));
```



#### 7. 如何构建自定义 Widgets

在Android中，您通常会继承View或已经存在的某个控件，然后覆盖其绘制方法来实现自定义View。

在Flutter中，一个自定义widget通常是通过组合其它widget来实现的，而不是继承。

我们来看看如何构建持有一个label的CustomButton。这是通过将Text与RaisedButton组合来实现的，而不是扩展RaisedButton并重写其绘制方法实现：

```dart
class CustomButton extends StatelessWidget {
  final String label;
  CustomButton(this.label);

  @override
  Widget build(BuildContext context) {
    return new RaisedButton(onPressed: () {}, child: new Text(label));
  }
}
```

Then you can use this CustomButton just like you would with any other Widget:

```dart
@override
  Widget build(BuildContext context) {
    return new Center(
      child: new CustomButton("Hello"),
    );
  }
}
```










