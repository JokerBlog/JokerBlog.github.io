---
layout:     post
title:      "Android性能优化篇之Bitmap优化"
subtitle:   ""
date:       2019-07-08 16:46:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### 介绍

我们知道Bitmap是内存溢出的主要原因之一。因为Bitmap特别占用内存，如果不及时是否就会导致内存溢出，应用崩溃。  

下面我们对Bitmap的是使用优化

#### 1.Bitmap 使用注意点

当我们需要获取图片的宽高等属性时且不对数据进行操作，那么我们不应该把图片的数据加载到内存中，这时我们可以设置inJustDecodeBounds属性为true.

```java
    BitmapFactory.Options opt=new BitmapFactory.Options();
    opt.inJustDecodeBounds=true;
    BitmapFactory.decodeFile(filePath, opt);
    final int height = options.outHeight;
    final int width = options.outWidth;
```

#### 2.减小Bitmap对象的内存占用

Bitmap是一个极容易消耗内存的大胖子，减小创建出来的Bitmap的内存占用是很重要的，通常来说有下面2个措施：  
1.inSampleSize：缩放比例，在把图片载入内存之前，我们需要先计算出一个合适的缩放比例，避免不必要的大图载入。  
2.decode format：解码格式，选择ARGB_8888/RBG_565/ARGB_4444/ALPHA_8，存在很大差异。  
我们来看下代码：

```java
    /**
 *   根据文件路径得到压缩的图片
 * @param filePath   文件路径
 * @param reqHeight  目标高
 * @param reqWidth   目标宽
 * @param config   options.inPreferredConfig=Config.RGB_565
 * @return
 */
public static Bitmap  getSmallImageForFile(String filePath,int reqHeight,int reqWidth,Config config){
    BitmapFactory.Options opt=new BitmapFactory.Options();
    opt.inJustDecodeBounds=true;
    BitmapFactory.decodeFile(filePath, opt);
    int inSampleSize = calacteInSampleSize(opt,reqHeight,reqWidth);
    opt.inPurgeable = true;
    opt.inInputShareable = true;
    opt.inPreferredConfig=config;
    opt.inJustDecodeBounds=false;
    Bitmap bitmap = BitmapFactory.decodeFile(filePath, opt);
    return  bitmap;
} 

/**
 * 计算缩放的值
 * @param options    BitmapFactory.Options
 * @param reqheight  目标高
 * @param reqWidth    目标宽
 */
public static int  calacteInSampleSize(BitmapFactory.Options options,int reqheight,int reqWidth){
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;
    if (width > reqWidth || height > reqheight) {
        if (width > height) {
            inSampleSize = Math.round((float) height / (float) reqheight);
        } else {
            inSampleSize = Math.round((float) width / (float) reqWidth);
        }
        final float totalPixels = width * height;
        final float maxTotalPixels = reqWidth * reqheight * 2;
        while (totalPixels / (inSampleSize * inSampleSize) > maxTotalPixels) {
            inSampleSize++;
        }
    }
    return inSampleSize;
}
```

使用

```java
    Bitmap smallBitmap = getSmallImageForFile(filePaht,480,320,Config.ARGB_
```

我们也可以对当前bitmap生产缩略图

```java
Bitmap thumbnailBitmap = ThumbnailUtils.extractThumbnail(bitmap, reqWidth, reqHeight, ThumbnailUtils.OPTIONS_RECYCLE_INPUT);
```

#### 3.Bitmap对象的复用

我们知道大多数对象的复用，最终实施的方案都是利用对象池技术，要么是在编写代码的时候显式的在程序里面去创建对象池，然后处理好复用的实现逻辑，要么就是利用系统框架既有的某些复用特性达到减少对象的重复创建，从而减少内存的分配与回收。 

 
在Bitmap中，利用inBitmap的高级特性提高Android系统在Bitmap分配与释放执行效率上的提升(3.0以及4.4以后存在一些使用限制上的差异)。使用inBitmap属性可以告知Bitmap解码器去尝试使用已经存在的内存区域，新解码的bitmap会尝试去使用之前那张bitmap在heap中所占据的pixel data内存区域，而不是去问内存重新申请一块区域来存放bitmap。利用这种特性，即使是上千张的图片，也只会仅仅只需要占用屏幕所能够显示的图片数量的内存大小。





![](https://upload-images.jianshu.io/upload_images/5748654-48ee33c314a08ec0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/444/format/webp)



###### 1).在SDK 11 -> 18之间，重用的bitmap大小必须是一致的，例如给inBitmap赋值的图片大小为100-100，那么新申请的bitmap必须也为100-100才能够被重用。从SDK 19开始，新申请的bitmap大小必须小于或者等于已经赋值过的bitmap大小

###### (2).新申请的bitmap与旧的bitmap必须有相同的解码格式，例如大家都是8888的，如果前面的bitmap是8888，那么就不能支持4444与565格式的bitmap了。 我们可以创建一个包含多种典型可重用bitmap的对象池，这样后续的bitmap创建都能够找到合适的“模板”去进行重用。

注意：在2.x的系统上，尽管bitmap是分配在native层，但是还是无法避免被计算到OOM的引用计数器里面。这里提示一下，不少应用会通过反射BitmapFactory.Options里面的inNativeAlloc来达到扩大使用内存的目的，但是如果大家都这么做，对系统整体会造成一定的负面影响，建议谨慎采纳。

### 4.Bitmap对象及时回收

虽然在大多数情况下，我们会对Bitmap增加缓存机制，但是在某些时候，部分Bitmap是需要及时回收的。例如临时创建的某个相对比较大的bitmap对象，在经过变换得到新的bitmap对象之后，应该尽快回收原始的bitmap，这样能够更快释放原始bitmap所占用的空间。



### 5.LRU管理Bitmap

LRU是Least Recently Used 近期最少使用算法。其实LruCache的作用就是对缓存的元素进行排序，当超过设定的内存值时就会将使用最少，使用最早元素先回收。  
使用Lru来管理Bitmap,设置最大内存，可以防止出现内存溢出。  
相关源码可以看[LruCache使用以及源码详细解析](https://blog.csdn.net/a847427920/article/details/52206165)



### 6.直接使用更小的图片



##### (1).我们可以在获取图片时告知服务器需要的图片的宽高, 以便服务器给出合适的图片, 避免浪费.

以七牛为例, 可以在请求图片的url中添加诸如质量, 格式, width, height等path来获取合适的图片资源.[七牛](https://developer.qiniu.com/dora/manual/1279/basic-processing-images-imageview2)



##### (2).使用本地图片压缩

关于图片压缩，将会在下一篇中讲解。






































