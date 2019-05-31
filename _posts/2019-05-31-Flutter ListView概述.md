---
layout:     post
title:      "Flutter ListView概述 "
subtitle:   ""
date:       2019-05-31 13:40:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - Flutter
---


在Flutter中，ListView就是一个ListView！

在Android ListView中，您可以创建一个适配器，然后您可以将它传递给ListView，该适配器将使用适配器返回的内容来展示每一行。 然而，你必须确保在合适的时机回收行，否则，你会得到各种疯狂的视觉和内存问题。

在Flutter中，由于Flutter的不可变的widget模型，将一个Widgets列表传递给的ListView，而Flutter将负责确保它们快速平滑地滚动。

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
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text("Sample App"),
      ),
      body: new ListView(children: _getListData()),
    );
  }

  _getListData() {
    List<Widget> widgets = [];
    for (int i = 0; i < 100; i++) {
      widgets.add(new Padding(padding: new EdgeInsets.all(10.0), child: new Text("Row $i")));
    }
    return widgets;
  }
}
```

### 2、 怎么知道哪个列表项被点击

在Android中，ListView有一个方法’onItemClickListener’来确定哪个列表项被点击。 Flutter中可以更轻松地通过您传入的处理回调来进行操作。

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
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text("Sample App"),
      ),
      body: new ListView(children: _getListData()),
    );
  }

  _getListData() {
    List<Widget> widgets = [];
    for (int i = 0; i < 100; i++) {
      widgets.add(new GestureDetector(
        child: new Padding(
            padding: new EdgeInsets.all(10.0),
            child: new Text("Row $i")),
        onTap: () {
          print('row tapped');
        },
      ));
    }
    return widgets;
  }
}
```

### 3、如何动态更新ListView

需要更新适配器并调用notifyDataSetChanged。在Flutter中，如果setState（）中更新widget列表，您会发现没有变化， 这是因为当setState被调用时，Flutter渲染引擎会遍历所有的widget以查看它们是否已经改变。 当遍历到你的ListView时，它会做一个`==`运算，以查看两个ListView是否相同，因为没有任何改变，因此没有更新数据。

要更新您的ListView，然后在setState中创建一个新的List（）并将所有旧数据复制到新列表中。这是实现更新的简单方法（译者语：此时状态改变，ListView被重新构建）

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
  List widgets = [];

  @override
  void initState() {
    super.initState();
    for (int i = 0; i < 100; i++) {
      widgets.add(getRow(i));
    }
  }

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text("Sample App"),
      ),
      body: new ListView(children: widgets),
    );
  }

  Widget getRow(int i) {
    return new GestureDetector(
      child: new Padding(
          padding: new EdgeInsets.all(10.0),
          child: new Text("Row $i")),
      onTap: () {
        setState(() {
          widgets = new List.from(widgets);
          widgets.add(getRow(widgets.length + 1));
          print('row $i');
        });
      },
    );
  }
}
```

然而，推荐的方法是使用ListView.Builder。当您拥有动态列表或包含大量数据的列表时，此方法非常有用。 这实际上相当于在Android上使用RecyclerView，它会自动为您回收列表元素：

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
  List widgets = [];

  @override
  void initState() {
    super.initState();
    for (int i = 0; i < 100; i++) {
      widgets.add(getRow(i));
    }
  }

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
        appBar: new AppBar(
          title: new Text("Sample App"),
        ),
        body: new ListView.builder(
            itemCount: widgets.length,
            itemBuilder: (BuildContext context, int position) {
              return getRow(position);
            }));
  }

  Widget getRow(int i) {
    return new GestureDetector(
      child: new Padding(
          padding: new EdgeInsets.all(10.0),
          child: new Text("Row $i")),
      onTap: () {
        setState(() {
          widgets.add(getRow(widgets.length + 1));
          print('row $i');
        });
      },
    );
  }
}
```

我们不是创建一个“新的ListView”，而是创建一个新的ListView.builder，它接受两个参数，即列表的初始长度和一个ItemBuilder函数。

ItemBuilder函数非常类似于Android适配器中的getView函数，它需要一个位置并返回要为该位置渲染的行。

最后，但最重要的是，如果您注意到onTap函数，在里面，我们不会再重新创建列表，而只是添加新元素到列表。




