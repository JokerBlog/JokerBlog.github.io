---
layout:     post
title:      "BootStrap3 "
subtitle:   "学习笔记"
date:       2019-02-25 11:26:05
author:     "Joker"
header-img: "img/bootstrap.png"
tags:
    - 前端
---

* container  80%

* container-fluid 100%

* 注意：不能互相嵌套

##### 2、排版

######## （1）标题

* class='h1 h2 h3...‘

######## （2）中心内容（突出）

* lead

* mark

######## （3）文字对齐

* text-left

* text-right

* text-center

##### 3、表格

* 响应式表格：table-responsive（重点）

  将任何`.table`元素包裹在`.table-responsive`元素内，即可创建响应式表格，其会在小屏幕设备上（小于768px）水平滚动。当屏幕大于 768px 宽度时，水平滚动条消失。

  ```html
  <div class="table-responsive">
    <table class="table">
      ...
    </table>
  </div>
  ```

* | Class      | 描述                 |
  | ---------- | ------------------ |
  | `.active`  | 鼠标悬停在行或单元格上时所设置的颜色 |
  | `.success` | 标识成功或积极的动作         |
  | `.info`    | 标识普通的提示信息或动作       |
  | `.warning` | 标识警告或需要用户注意        |
  | `.danger`  | 标识危险或潜在的带来负面影响的动作  |

```html
<!-- On rows -->
<tr class="active">...</tr>
<tr class="success">...</tr>
<tr class="warning">...</tr>
<tr class="danger">...</tr>
<tr class="info">...</tr>

<!-- On cells (`td` or `th`) -->
<tr>
  <td class="active">...</td>
  <td class="success">...</td>
  <td class="warning">...</td>
  <td class="danger">...</td>
  <td class="info">...</td>
</tr>
```

##### 4、表单

* input需要配合table使用

* 横向排列的表单：form-inline

  为`<form>`元素添加`.form-inline`类可使其内容左对齐并且表现为`inline-block`级别的控件。**只适用于视口（viewport）至少在 768px 宽度时（视口宽度再小的话就会使表单折叠）。**

  ![Snip20190222_79](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_79.png)

* 水平排列的表单：form-horizontal

  通过为表单添加`.form-horizontal`类，并联合使用 Bootstrap 预置的栅格类，可以将`label`标签和控件组水平并排布局。这样做将改变`.form-group`的行为，使其表现为栅格系统中的行（row），因此就无需再额外添加`.row`了。

  ```html
  <form class="form-horizontal">
    <div class="form-group">
      <label for="inputEmail3" class="col-sm-2 control-label">Email</label>
      <div class="col-sm-10">
        <input type="email" class="form-control" id="inputEmail3" placeholder="Email">
      </div>
    </div>
    <div class="form-group">
      <label for="inputPassword3" class="col-sm-2 control-label">Password</label>
      <div class="col-sm-10">
        <input type="password" class="form-control" id="inputPassword3" placeholder="Password">
      </div>
    </div>
    <div class="form-group">
      <div class="col-sm-offset-2 col-sm-10">
        <div class="checkbox">
          <label>
            <input type="checkbox"> Remember me
          </label>
        </div>
      </div>
    </div>
    <div class="form-group">
      <div class="col-sm-offset-2 col-sm-10">
        <button type="submit" class="btn btn-default">Sign in</button>
      </div>
    </div>
  </form>
  ```

* input控制：form-control

  单独的表单控件会被自动赋予一些全局样式。所有设置了`.form-control`类的`<input>`、`<textarea>`和`<select>`元素都将被默认设置宽度属性为`width: 100%;`。 将`label`元素和前面提到的控件包裹在`.form-group`中可以获得最好的排列。

  ![Snip20190222_80](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_80.png)

* 连接：input-group

  ![Snip20190222_81](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_81.png)

##### 5、按钮

* 尺寸：lg、sm、xs

![Snip20190222_82](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_82.png)

##### 6、图片

* 响应式图片：img-responsive

  在 Bootstrap 版本 3 中，通过为图片添加`.img-responsive`类可以让图片支持响应式布局。其实质是为图片设置了`max-width: 100%;`、`height: auto;`和`display: block;`属性，从而让图片在其父元素中更好的缩放。

  如果需要让使用了`.img-responsive`类的图片水平居中，请使用`.center-block`类，不要用`.text-center`。

  ```html
  <img src="..." class="img-responsive" alt="Responsive image">
  ```

* 图片居中：center-block

##### 7、辅助类

* 浮动

左浮动：pull-left

右浮动：pull-right

通过添加一个类，可以将任意元素向左或向右浮动。`!important`被用来明确 CSS 样式的优先级。这些类还可以作为 mixin（参见 less 文档） 使用。

```html
<div class="pull-left">...</div>
<div class="pull-right">...</div>
```

* 清除浮动

**通过为父元素**添加`.clearfix`类可以很容易地清除浮动（`float`）。这里所使用的是 Nicolas Gallagher 创造的[micro clearfix](http://nicolasgallagher.com/micro-clearfix-hack/)方式。此类还可以作为 mixin 使用。

```html
<!-- Usage as a class -->
<div class="clearfix">...</div>
```

* 居中：center-block

  为任意元素设置`display: block`属性并通过`margin`属性让其中的内容居中。下面列出的类还可以作为 mixin 使用。

  ```html
  <div class="center-block">...</div>
  ```

* 显示与隐藏（show与hidden）

  `.show`和`.hidden`类可以强制任意元素显示或隐藏(**对于屏幕阅读器也能起效**)。这些类通过`!important`来避免 CSS 样式优先级问题，就像[quick floats](https://v3.bootcss.com/css/#helper-classes-floats)一样的做法。注意，这些类只对块级元素起作用，另外，还可以作为 mixin 使用。

  `.hide`类仍然可用，但是它不能对屏幕阅读器起作用，并且从 v3.0.1 版本开始就**不建议使用**了。请使用`.hidden`或`.sr-only`。

  另外，`.invisible`类可以被用来仅仅影响元素的可见性，也就是说，元素的`display`属性不被改变，并且这个元素仍然能够影响文档流的排布。

  ```html
  <div class="show">...</div>
  <div class="hidden">...</div>
  ```

##### 8、响应式工具

![Snip20190222_83](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_83.png)

##### 9、栅格系统（参数、布局）

Bootstrap 提供了一套响应式、移动设备优先的流式栅格系统，随着屏幕或视口（viewport）尺寸的增加，系统会自动分为最多12列。它包含了易于使用的[预定义类](https://v3.bootcss.com/css/#grid-example-basic)，还有强大的[mixin 用于生成更具语义的布局](https://v3.bootcss.com/css/#grid-less)。

栅格系统用于通过一系列的行（row）与列（column）的组合来创建页面布局，你的内容就可以放入这些创建好的布局中。下面就介绍一下 Bootstrap 栅格系统的工作原理：

- “行（row）”必须包含在`.container`（固定宽度）或`.container-fluid`（100% 宽度）中，以便为其赋予合适的排列（aligment）和内补（padding）。
- 通过“行（row）”在水平方向创建一组“列（column）”。
- 你的内容应当放置于“列（column）”内，并且，只有“列（column）”可以作为行（row）”的直接子元素。
- 类似`.row`和`.col-xs-4`这种预定义的类，可以用来快速创建栅格布局。Bootstrap 源码中定义的 mixin 也可以用来创建语义化的布局。
- 通过为“列（column）”设置`padding`属性，从而创建列与列之间的间隔（gutter）。通过为`.row`元素设置负值`margin`从而抵消掉为`.container`元素设置的`padding`，也就间接为“行（row）”所包含的“列（column）”抵消掉了`padding`。
- 负值的 margin就是下面的示例为什么是向外突出的原因。在栅格列中的内容排成一行。
- 栅格系统中的列是通过指定1到12的值来表示其跨越的范围。例如，三个等宽的列可以使用三个`.col-xs-4`来创建。
- 如果一“行（row）”中包含了的“列（column）”大于 12，多余的“列（column）”所在的元素将被作为一个整体另起一行排列。
- 栅格类适用于与屏幕宽度大于或等于分界点大小的设备 ， 并且针对小屏幕设备覆盖栅格类。 因此，在元素上应用任何`.col-md-*`栅格类适用于与屏幕宽度大于或等于分界点大小的设备 ， 并且针对小屏幕设备覆盖栅格类。 因此，在元素上应用任何`.col-lg-*`不存在， 也影响大屏幕设备。

![Snip20190222_84](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_84.png)

![Snip20190222_85](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_85.png)

* 尺寸

  lg：大于1200px（大桌面显示器）

  md:大于992px（桌面显示器）

  sm:大于等于768px（平板）

  xs：小于768px（手机）

##### 10、栅格系统（偏移、嵌套、排序）

* 列偏移

  使用`.col-md-offset-*`类可以将列向右侧偏移。这些类实际是通过使用`*`选择器为当前元素增加了左侧的边距（margin）。例如，`.col-md-offset-4`类将`.col-md-4`元素向右侧偏移了4个列（column）的宽度。

![Snip20190222_86](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_86.png)

* 嵌套列

  为了使用内置的栅格系统将内容再次嵌套，可以通过添加一个新的`.row`元素和一系列`.col-sm-*`元素到已经存在的`.col-sm-*`元素内。被嵌套的行（row）所包含的列（column）的个数不能超过12（其实，没有要求你必须占满12列）。

  ![Snip20190222_88](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_88.png)

* 列排序

  通过使用`.col-md-push-*`和`.col-md-pull-*`类就可以很容易的改变列（column）的顺序。

![Snip20190222_89](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_89.png)





#### 11、字体图标

出于性能的考虑，所有图标都需要一个基类和对应每个图标的类。把下面的代码放在任何地方都可以正常使用。注意，为了设置正确的内补（padding），务必在图标和文本之间添加一个空格。

*  不要和其他组件混合使用

图标类不能和其它组件直接联合使用。它们不能在同一个元素上与其他类共同存在。应该创建一个嵌套的`<span>`标签，并将图标类应用到这个`<span>`标签上。

*  只对内容为空的元素起作用

图标类只能应用在不包含任何文本内容或子元素的元素上。



![Snip20190222_90](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_90.png)



##### 12、下拉菜单

将下拉菜单触发器和下拉菜单都包裹在`.dropdown`里，或者另一个声明了`position: relative;`的元素。然后加入组成菜单的 HTML 代码。

![Snip20190222_91](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_91.png)

* 点击谁：需要给元素加入自定义属性

  ```html
  data-toggle="dropdown"
  ```

* 显示谁：加入类（class）

  ```html
  class="dropdown-menu"
  ```



#### 13、按钮组

通过按钮组容器把一组按钮放在同一行里。通过与[按钮插件](https://v3.bootcss.com/javascript/#buttons)联合使用，可以设置为单选框或多选框的样式和行为。

![Snip20190222_92](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_92.png)



* 尺寸

  只要给`.btn-group`加上`.btn-group-*`类，就省去为按钮组中的每个按钮都赋予尺寸类了，如果包含了多个按钮组时也适用。

![Snip20190222_94](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_94.png)



* 嵌套

  想要把下拉菜单混合到一系列按钮中，只须把`.btn-group`放入另一个`.btn-group`中。

  ![Snip20190222_95](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_95.png)



* 垂直排列

  让一组按钮垂直堆叠排列显示而不是水平排列。**分列式按钮下拉菜单不支持这种方式。**

![Snip20190222_96](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_96.png)



*  两端对齐排列的按钮组

让一组按钮拉长为相同的尺寸，填满父元素的宽度。对于按钮组中的按钮式下拉菜单也同样适用。

 

关于边框的处理

由于对两端对齐的按钮组使用了特定的 HTML 和 CSS （即`display: table-cell`），两个按钮之间的边框叠加在了一起。在普通的按钮组中，`margin-left: -1px`用于将边框重叠，而没有删除任何一个按钮的边框。然而，`margin`属性不支持`display: table-cell`。因此，根据你对 Bootstrap 的定制，你可以删除或重新为按钮的边框设置颜色。



 IE8 和边框

Internet Explorer 8 不支持在两端对齐的按钮组中绘制边框，无论是`<a>`或`<button>`元素。为了照顾 IE8，把每个按钮放入另一个`.btn-group`中即可。

参见[#12476](https://github.com/twbs/bootstrap/issues/12476)获取详细信息。



只须将一系列`.btn`元素包裹到`.btn-group.btn-group-justified`中即可。



![Snip20190222_98](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_98.png)







#### 14、输入框组

通过在文本输入框`<input>`前面、后面或是两边加上文字或按钮，可以实现对表单控件的扩展。为`.input-group`赋予`.input-group-addon`或`.input-group-btn`类，可以给`.form-control`的前面或后面添加额外的元素。



* 只支持文本输入框`<input>`

这里请避免使用`<select>`元素，因为 WebKit 浏览器不能完全绘制它的样式。

避免使用`<textarea>`元素，由于它们的`rows`属性在某些情况下不被支持。

*  输入框组中的工具提示和弹出框需要特别的设置

  

为`.input-group`中所包含的元素应用工具提示（tooltip）或popover（弹出框）时，必须设置`container: 'body'`参数，为的是避免意外的副作用（例如，工具提示或弹出框被激活后，可能会让当前元素变得更宽或/和变得失去其圆角）。



*  不要和其他组件混用

不要将表单组或栅格列（column）类直接和输入框组混合使用。而是将输入框组嵌套到表单组或栅格相关元素的内部。



![Snip20190222_99](/Users/dyonline/Documents/中基凌云/文档/Snip20190222_99.png)



#### 15、导航

Bootstrap 中的导航组件都依赖同一个`.nav`类，状态类也是共用的。改变修饰类可以改变样式。

*  在标签页上使用导航需要依赖 JavaScript 标签页插件

由于标签页需要控制内容区的展示，因此，你必须使用[标签页组件的 JavaScript 插件](https://v3.bootcss.com/javascript/#tabs)。另外还要添加`role`和 ARIA 属性 – 详细信息请参考该插件的[实例](https://v3.bootcss.com/javascript/#tabs-usage)。

*  确保导航组件的可访问性

如果你在使用导航组件实现导航条功能，务必在`<ul>`的最外侧的逻辑父元素上添加`role="navigation"`属性，或者用一个`<nav>`元素包裹整个导航组件。不要将 role 属性添加到`<ul>`上，因为这样可以被辅助设备（残疾人用的）上被识别为一个真正的列表。



注意`.nav-tabs`类依赖`.nav`基类。

```html
<ul class="nav nav-tabs">
  <li role="presentation" class="active"><a href="#">Home</a></li>
  <li role="presentation"><a href="#">Profile</a></li>
  <li role="presentation"><a href="#">Messages</a></li>
</ul>
```



#### 16、导航条

* 默认样式的导航条

导航条是在您的应用或网站中作为导航页头的响应式基础组件。它们在移动设备上可以折叠（并且可开可关），且在视口（viewport）宽度增加时逐渐变为水平展开模式。

* 两端对齐的导航条导航链接已经被弃用了。

*  导航条内所包含元素溢出

由于 Bootstrap 并不知道你在导航条内放置的元素需要占据多宽的空间，你可能会遇到导航条中的内容折行的情况（也就是导航条占据两行）。解决办法如下：

1. 减少导航条内所有元素所占据的宽度。
2. 在某些尺寸的屏幕上（利用[响应式工具类](https://v3.bootcss.com/css/#responsive-utilities)）隐藏导航条内的一些元素。
3. 修改导航条在水平排列和折叠排列互相转化时，触发这个转化的最小屏幕宽度值。可以通过修改`@grid-float-breakpoint`变量实现，或者自己重写相关的媒体查询代码，覆盖 Bootstrap 的默认值。



将导航条内放置品牌标志的地方替换为`<img>`元素即可展示自己的品牌图标。由于`.navbar-brand`已经被设置了内补（padding）和高度（height），你需要根据自己的情况添加一些 CSS 代码从而覆盖默认设置。

[![Brand](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACgAAAAoCAYAAACM/rhtAAAB+0lEQVR4AcyYg5LkUBhG+1X2PdZGaW3btm3btm3bHttWrPomd1r/2Jn/VJ02TpxcH4CQ/dsuazWgzbIdrm9dZVd4pBz4zx2igTaFHrhvjneVXNHCSqIlFEjiwMyyyOBilRgGSqLNF1jnwNQdIvAt48C3IlBmHCiLQHC2zoHDu6zG1iXn6+y62ScxY9AODO6w0pvAqf23oSE4joOfH6OxfMoRnoGUm+de8wykbFt6wZtA07QwtNOqKh3ZbS3Wzz2F+1c/QJY0UCJ/J3kXWJfv7VhxCRRV1jGw7XI+gcO7rEFFRvdYxydwcPsVsC0bQdKScngt4iUTD4Fy/8p7PoHzRu1DclwmgmiqgUXjD3oTKHbAt869qdJ7l98jNTEblPTkXMwetpvnftA0LLHb4X8kiY9Kx6Q+W7wJtG0HR7fdrtYz+x7iya0vkEtUULIzCjC21wY+W/GYXusRH5kGytWTLxgEEhePPwhKYb7EK3BQuxWwTBuUkd3X8goUn6fMHLyTT+DCsQdAEXNzSMeVPAJHdF2DmH8poCREp3uwm7HsGq9J9q69iuunX6EgrwQVObjpBt8z6rdPfvE8kiiyhsvHnomrQx6BxYUyYiNS8f75H1w4/ISepDZLoDhNJ9cdNUquhRsv+6EP9oNH7Iff2A9g8h8CLt1gH0Qf9NMQAFnO60BJFQe0AAAAAElFTkSuQmCC)](https://v3.bootcss.com/components/#)

```html
<nav class="navbar navbar-default">
  <div class="container-fluid">
    <div class="navbar-header">
      <a class="navbar-brand" href="#">
        <img alt="Brand" src="...">
      </a>
    </div>
  </div>
</nav>
```




