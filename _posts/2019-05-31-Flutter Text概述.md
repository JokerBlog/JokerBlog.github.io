---
layout:     post
title:      "Flutter Text概述 "
subtitle:   ""
date:       2019-05-31 13:49:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - Flutter
---

### 1、如何在 Text widget上设置自定义字体

在Android SDK（从Android O开始）中，创建一个Font资源文件并将其传递到TextView的FontFamily参数中。

在Flutter中，首先你需要把你的字体文件放在项目文件夹中（最好的做法是创建一个名为assets的文件夹）

接下来在pubspec.yaml文件中，声明字体：

```yaml
fonts:
   - family: MyCustomFont
     fonts:
       - asset: fonts/MyCustomFont.ttf
       - style: italic
```

最后，将字体应用到Text widget:

```dart
@override
Widget build(BuildContext context) {
  return new Scaffold(
    appBar: new AppBar(
      title: new Text("Sample App"),
    ),
    body: new Center(
      child: new Text(
        'This is a custom font text',
        style: new TextStyle(fontFamily: 'MyCustomFont'),
      ),
    ),
  );
}
```

### 2、 如何在Text上定义样式

除了自定义字体，您还可在Text上自定义很多不同的样式

Text的样式参数需要一个TextStyle对象，您可以在其中自定义许多参数，如

- color
- decoration
- decorationColor
- decorationStyle
- fontFamily
- fontSize
- fontStyle
- fontWeight
- hashCode
- height
- inherit
- letterSpacing
- textBaseline
- wordSpacing


















