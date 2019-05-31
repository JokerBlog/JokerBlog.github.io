---
layout:     post
title:      "Flutter Layout概述 "
subtitle:   ""
date:       2019-05-31 09:48:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - Flutter
---


在Android中，使用LinearLayout来使您的控件呈水平或垂直排列。在Flutter中，您可以使用Row或Co​​lumn来实现相同的结果。

```dart
@override
Widget build(BuildContext context) {
  return new Row(
    mainAxisAlignment: MainAxisAlignment.center,
    children: <Widget>[
      new Text('Row One'),
      new Text('Row Two'),
      new Text('Row Three'),
      new Text('Row Four'),
    ],
  );
}
```

```dart
@override
Widget build(BuildContext context) {
  return new Column(
    mainAxisAlignment: MainAxisAlignment.center,
    children: <Widget>[
      new Text('Column One'),
      new Text('Column Two'),
      new Text('Column Three'),
      new Text('Column Four'),
    ],
  );
}
```

### 2、RelativeLayout在Flutter中等价于什么

RelativeLayout用于使widget相对于彼此位置排列。在Flutter中，有几种方法可以实现相同的结果

您可以通过使用Column、Row和Stack的组合来实现RelativeLayout的效果。您可以为widget构造函数指定相对于父组件的布局规则。

一个在Flutter中构建RelativeLayout的好例子，请参考在StackOverflow上

[https://stackoverflow.com/questions/44396075/equivalent-of-relativelayout-in -flutter](https://stackoverflow.com/questions/44396075/equivalent-of-relativelayout-in-flutter)



### 3、ScrollView在Flutter中等价于什么

在Android中，ScrollView允许您包含一个子控件，以便在用户设备的屏幕比控件内容小的情况下，使它们可以滚动。

在Flutter中，最简单的方法是使用ListView。但在Flutter中，一个ListView既是一个ScrollView，也是一个Android ListView。

```dart
@override
Widget build(BuildContext context) {
  return new ListView(
    children: <Widget>[
      new Text('Row One'),
      new Text('Row Two'),
      new Text('Row Three'),
      new Text('Row Four'),
    ],
  );
}
```




