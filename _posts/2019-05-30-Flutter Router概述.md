---
layout:     post
title:      "Flutter Router概述 "
subtitle:   ""
date:       2019-05-30 15:32:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - Flutter
---


在Android中，Intents主要有两种使用场景：在Activity之间切换，以及调用外部组件。 Flutter不具有Intents的概念，但如果需要的话，Flutter可以通过Native整合来触发Intents。

要在Flutter中切换屏幕，您可以访问路由以绘制新的Widget。 管理多个屏幕有两个核心概念和类：Route 和 Navigator。Route是应用程序的“屏幕”或“页面”的抽象（可以认为是Activity）， Navigator是管理Route的Widget。Navigator可以通过push和pop route以实现页面切换。

和Android相似，您可以在AndroidManifest.xml中声明您的Activities，在Flutter中，您可以将具有指定Route的Map传递到顶层MaterialApp实例

```dart
void main() {
  runApp(new MaterialApp(
    home: new MyAppHome(), // becomes the route named '/'
    routes: <String, WidgetBuilder> {
      '/a': (BuildContext context) => new MyPage(title: 'page A'),
      '/b': (BuildContext context) => new MyPage(title: 'page B'),
      '/c': (BuildContext context) => new MyPage(title: 'page C'),
    },
  ));
}
```

然后，您可以通过Navigator来切换到命名路由的页面。

```dart
Navigator.of(context).pushNamed('/b');
```

Intents的另一个主要的用途是调用外部组件，如Camera或File picker。为此，您需要和native集成（或使用现有的库）

参阅 [Flutter Plugins] 了解如何与native集成.

### 2、如何在Flutter中处理来自外部应用程序传入的Intents

在Flutter中，您可以在渲染Flutter视图时请求数据。

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

void main() {
  runApp(new SampleApp());
}

class SampleApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'Sample Shared App Handler',
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
  static const platform = const MethodChannel('app.channel.shared.data');
  String dataShared = "No data";

  @override
  void initState() {
    super.initState();
    getSharedText();
  }

  @override
  Widget build(BuildContext context) {
    return new Scaffold(body: new Center(child: new Text(dataShared)));
  }

  getSharedText() async {
    var sharedData = await platform.invokeMethod("getSharedText");
    if (sharedData != null) {
      setState(() {
        dataShared = sharedData;
      });
    }
  }
}
```

### 3、startActivityForResult 在Flutter中等价于什么

处理Flutter中所有路由的Navigator类可用于从已经push到栈的路由中获取结果。 这可以通过等待push返回的Future来完成。例如，如果您要启动让用户选择其位置的位置的路由，则可以执行以下操作：

```dart
Map coordinates = await Navigator.of(context).pushNamed('/location');
```

然后在你的位置路由中，一旦用户选择了他们的位置，你可以将结果”pop”出栈

```dart
Navigator.of(context).pop({"lat":43.821757,"long":-79.226392});
```






