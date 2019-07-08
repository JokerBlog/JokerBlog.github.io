---
layout:     post
title:      "Android NDK开发之旅--Apk增量更新"
subtitle:   ""
date:       2019-07-05 14:11:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
- Android     
---

### 前言

有关APK更新的技术比较多，例如：增量更新、插件式开发、热修复、RN、静默安装。  
下面简单介绍一下：

| 更新方式  | 签名                                                                            |
| ----- | ----------------------------------------------------------------------------- |
| 增量更新  | 旧版本Apk（v1.0）和新（v2.0）、旧版本Apk（v1.0）生成的差分包（apk.patch 质量小） |合并成为新版本Apk（v2.0）安装。 | |
| 插件式开发 | 给宿主APK提供插件，扩展（需要的时候再下载），可以动态地替换。主要技术是动态加载的知识。                                 |
| 热修复   | 通过NDK底层去修复，也是C/C++的技术。                                                        |
| RN    | 通过JS脚本去修复APK。                                                                 |
| 静默安装  | 需要root权限，适配不同手机ROM很麻烦。                                                        |

### 什么是增量更新？

增量更新就是原有app的基础上只更新发生变化的地方，其余保持原样。  
与原来每次更新都要下载完整apk包的做法相比，这样做的好处显而易见：每次变化的地方总是比较少，因此更新包的体积就会小很多。



### 增量更新的流程

1.APP检测最新版本：把当前版本告诉服务端，服务端进行判断。  
如果有新版本，服务端需要对当前版本的APK与最新版本的APK进行一次差分，产生patch差分文件。（或者新版本的APK上传到服务端的时候就已经差分好了）  

2.APP在后台下载差分文件，进行文件的MD5校验，在本地进行合并（跟本地的data目录下面的APK文件合并），合并出最新的APK之后，提示用户安装。  

3.增量更新的最终目的：省流量地更新宿主APK。

差分的处理比较麻烦的地方就是要针对不同的应用市场渠道和众多不同版本进行差分。

注意：新版本有可能比旧版本小，差分只是把变化的部分记录下来。

![](https://upload-images.jianshu.io/upload_images/1824809-4a1f65020badfa47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/828/format/webp)



### 服务器端行为（后台工程师操作）

##### 1.下载拆分和合并要用的第三方库（bsdiff、bzip2）

我们使用到的第三方库是：Binary diff，简称bsdiff，这个库专门用来实现文件的差分和合并的，它的官网如下：  
[http://www.daemonology.net/bsdiff/](https://link.jianshu.com?t=http%3A%2F%2Fwww.daemonology.net%2Fbsdiff%2F)

![](https://upload-images.jianshu.io/upload_images/1824809-6faa47e1a0fa6f6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



在这里我们可以点击文中的"here"下载源码，这是Linux源码。也可以下载Windows版本的源码，点击"Windows port"。

##### 建议Windows 下用sbsdiff4.3-win32-src编译

这个库引用了bzip2这个库，官网如下：  
[http://www.bzip.org/](https://link.jianshu.com?t=http%3A%2F%2Fwww.bzip.org%2F)



##### 2.编译第三方库源码生成dll动态库

##### 为了方便演示，我在Windows 10平台下用VS2017编译，实际情况服务器大都在Linux系统下运行，这个大家去测试吧。

##### Windows 下生成dll动态库参考

![](https://upload-images.jianshu.io/upload_images/1824809-896e38e2de3d2296.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/436/format/webp)



##### 注意：com_haocai_bsdiff_BsDiff.h 是根据Java文件声明得到的，步骤省略。

##### 编译过程中会有以下错误提示

- 字符集问题
- 用了不安全和过时的函数
- SDL检查不通过

##### 以下是解决办法：

![](https://upload-images.jianshu.io/upload_images/1824809-d4c13e2c27dc3448.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

![](https://upload-images.jianshu.io/upload_images/1824809-4bd5576358d58b75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

![](https://upload-images.jianshu.io/upload_images/1824809-5ee4d8c5ffdfc502.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



另外，可能报头文件找不到的错误，这有可能是编码问题，因为外国人使用的苹果电脑跟Windows电脑的编译不一致产生的。可以通过Notepad++的转码功能进行转码，全部转为UTF-8无BOM格式编码即可，Windows、Linux通用的。

我们项目属性里面的生成配置里面选择DLL，并且修改解决方案为你的电脑的对应平台，然后编译，生成DLL动态库文件。



#### 3.Java代码调用

创建Web项目，用来做APP的服务端。创建工具类专门用于产生差分包：

```java
public class BsDiff {
    /**
     * 差分
     * @param oldfile
     * @param newfile
     * @param patchfile
     */
    public native static void diff(String oldfile,String newfile,String patchfile);

    static {
        System.loadLibrary("bsdiff");
    }
}
```

```c
JNIEXPORT void JNICALL Java_com_haocai_bsdiff_BsDiff_diff
(JNIEnv *env, jclass jcls, jstring oldfile_jstr, jstring newfile_jstr, jstring patchfile_jstr) {
    int argc = 4;
    char* oldfile = (char*)env->GetStringUTFChars(oldfile_jstr, NULL);
    char* newfile = (char*)env->GetStringUTFChars(newfile_jstr, NULL);
    char* patchfile = (char*)env->GetStringUTFChars(patchfile_jstr, NULL);

    //参数(第一个参数无效)
    char *argv[4];
    argv[0] = { "bsdiff" };
    argv[1] = oldfile;
    argv[2] = newfile;
    argv[3] = patchfile;

    bsdiff_main(argc, argv);

    env->ReleaseStringUTFChars(oldfile_jstr, oldfile);
    env->ReleaseStringUTFChars(newfile_jstr, newfile);
    env->ReleaseStringUTFChars(patchfile_jstr, patchfile);
};
```

通过研究bsdiff的源码，我们发现bsdiff.cpp里面的main函数就是入口函数，避免歧义把函数名main改为bsdiff_main，然后通过JNI去调用。  

##### 根据bsdiff.cpp中bsdiff_main函数方法中有以下关键语句

```c
if (argc != 4) errx(1, "usage: %s oldfile newfile patchfile\n", argv[0]);
```

然后我们准备两个APK文件，不同版本的，最好Java代码、资源都不一样。

写一个Java测试类生成差分包：

```java
package com.haocai.bsdiff;

public class ConstantsWin {
    
    //路径不能包含中文
    public static final String OLD_APK_PATH = "D:/android_apks/test_old.apk";
    
    public static final String NEW_APK_PATH = "D:/android_apks/test_new.apk";
    
    public static final String PATCH_PATH = "D:/android_apks/apk.patch";
}
```

```java
package com.haocai.bsdiff;

/**
 * Created by Administrator on 2017/11/14.
 */
public class BsDiffTest {
    public static void main(String[] args){
        //得到差分包
        BsDiff.diff(ConstantsWin.OLD_APK_PATH,ConstantsWin.NEW_APK_PATH,ConstantsWin.PATCH_PATH);
    }
}
```

##### 生成结果如下图所示：

![](https://upload-images.jianshu.io/upload_images/1824809-f95ad0d098885de9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/653/format/webp)



##### 注意：

- ###### test_new.apk、test_old.apk 要先放在目标目录

- ###### bsdiff.cpp中生成差分包的程序方法是异步的，所以生成完整的apk.patch可能要等一下。apk.patch体积大小停止增长，表示生成结束。

#### 4.简单搭建后台JavaWeb供Android前端下载apk.patch差分包

#### 参考 [Intellij idea创建javaWeb以及Servlet简单实现](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fyhao2014%2Farticle%2Fdetails%2F45740111)

![](https://upload-images.jianshu.io/upload_images/1824809-ed1d30729d6ad453.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/464/format/webp)



###### 在浏览器中输入

[http://localhost:8080/App_Update_Web/patchfile/apk.patch](https://link.jianshu.com/?t=http%3A%2F%2Flocalhost%3A8080%2FApp_Update_Web%2Fpatchfile%2Fapk.patch)

由于Android手机本来就是Linux系统，因此我们直接使用bsdiff的Linux版本的库即可。  
跟服务器端一样，在这里我们把bspatch.c中的main函数改为bspatch_main，提供JNI调用：

```c
//合并
JNIEXPORT void JNICALL Java_com_haocai_app_update_BsPatch_patch
  (JNIEnv *env, jclass jcls, jstring oldfile_jstr, jstring newfile_jstr, jstring patchfile_jstr){
    int argc = 4;
    char* oldfile = (char*)(*env)->GetStringUTFChars(env,oldfile_jstr, NULL);
    char* newfile = (char*)(*env)->GetStringUTFChars(env,newfile_jstr, NULL);
    char* patchfile = (char*)(*env)->GetStringUTFChars(env,patchfile_jstr, NULL);

    //参数（第一个参数无效）
    char *argv[4];
    argv[0] = "bspatch";
    argv[1] = oldfile;
    argv[2] = newfile;
    argv[3] = patchfile;

    bspatch_main(argc,argv);

    (*env)->ReleaseStringUTFChars(env,oldfile_jstr, oldfile);
    (*env)->ReleaseStringUTFChars(env,newfile_jstr, newfile);
    (*env)->ReleaseStringUTFChars(env,patchfile_jstr, patchfile);

  }
```

代码v1.0差分包合并核心代码如下：

```java
//合并
JNIEXPORT void JNICALL Java_com_haocai_app_update_BsPatch_patch
  (JNIEnv *env, jclass jcls, jstring oldfile_jstr, jstring newfile_jstr, jstring patchfile_jstr){
    int argc = 4;
    char* oldfile = (char*)(*env)->GetStringUTFChars(env,oldfile_jstr, NULL);
    char* newfile = (char*)(*env)->GetStringUTFChars(env,newfile_jstr, NULL);
    char* patchfile = (char*)(*env)->GetStringUTFChars(env,patchfile_jstr, NULL);

    //参数（第一个参数无效）
    char *argv[4];
    argv[0] = "bspatch";
    argv[1] = oldfile;
    argv[2] = newfile;
    argv[3] = patchfile;

    bspatch_main(argc,argv);

    (*env)->ReleaseStringUTFChars(env,oldfile_jstr, oldfile);
    (*env)->ReleaseStringUTFChars(env,newfile_jstr, newfile);
    (*env)->ReleaseStringUTFChars(env,patchfile_jstr, patchfile);

  }


代码v1.0差分包合并核心代码如下：
package com.haocai.app.update;

import android.Manifest;
import android.content.pm.PackageManager;
import android.os.Handler;
import android.os.Message;
import android.support.annotation.NonNull;
import android.support.annotation.Nullable;
import android.support.v4.app.ActivityCompat;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.text.format.Formatter;
import android.widget.Toast;
import com.lzy.okgo.OkGo;
import com.lzy.okgo.callback.FileCallback;
import com.lzy.okgo.model.Progress;
import com.lzy.okgo.model.Response;
import com.lzy.okgo.request.base.Request;
import java.io.File;
import java.text.NumberFormat;

public class MainActivity extends AppCompatActivity {

    private static final int REQUEST_PERMISSION_STORAGE = 0x01;
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case 0:
                    Toast.makeText(MainActivity.this, "您正在进行省流量更新", Toast.LENGTH_SHORT).show();
                    ApkUtils.installApk(MainActivity.this, Constants.NEW_APK_PATH);
                    break;
            }
        }
    };
    private NumberFormat numberFormat;


    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        setTitle("简单文件下载");

        numberFormat = NumberFormat.getPercentInstance();
        numberFormat.setMinimumFractionDigits(2);

        checkSDCardPermission();

        /**
         * 因为后台没有写版本判断语句
         * 在高版本下暂时先注释fileDownload(); 否则一直下载安装
         *
         * 低版本下运行fileDownload();
         */
         fileDownload();


    }


    /**
     * 检查SD卡权限
     */
    protected void checkSDCardPermission() {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE) != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE}, REQUEST_PERMISSION_STORAGE);
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == REQUEST_PERMISSION_STORAGE) {
            if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                //获取权限
                fileDownload();
            } else {
                Toast.makeText(getApplicationContext(), "权限被禁止，无法下载文件！", Toast.LENGTH_SHORT).show();
            }
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        //Activity销毁时，取消网络请求
        OkGo.getInstance().cancelTag(this);
    }


    public void fileDownload() {

        OkGo.<File>get(Constants.URL_PATCH_DOWNLOAD)//
                .tag(this)//
                .execute(new FileCallback(Constants.SD_CARD, Constants.PATCH_FILE) {

                    @Override
                    public void onStart(Request<File, ? extends Request> request) {
                    }

                    @Override
                    public void onSuccess(Response<File> response) {

                        new Thread(new Runnable() {
                            @Override
                            public void run() {

                                try {
                                    //      File patchFile = new File(Constants.SD_CARD, Constants.PATCH_FILE);
                                    String oldfile = ApkUtils.getSourceApkPath(MainActivity.this, getPackageName());
                                    String newfile = Constants.NEW_APK_PATH;
                                    String patchfile = Constants.SD_CARD + File.separator + Constants.PATCH_FILE;
                                    BsPatch.patch(oldfile, newfile, patchfile);

                                    mHandler.sendEmptyMessage(0);
                                } catch (Exception e) {
                                    e.printStackTrace();
                                }
                            }
                        }).start();


                    }

                    @Override
                    public void onError(Response<File> response) {

                    }

                    @Override
                    public void downloadProgress(Progress progress) {
                        System.out.println(progress);

                        String downloadLength = Formatter.formatFileSize(getApplicationContext(), progress.currentSize);
                        String totalLength = Formatter.formatFileSize(getApplicationContext(), progress.totalSize);
                        String speed = Formatter.formatFileSize(getApplicationContext(), progress.speed);
                        System.out.println(downloadLength);
                    }
                });
    }

}


```

主要的逻辑在fileDownload方法中，我们先下载差分包，然后在本地合成，最后提示用户安装。

为了达到明显的效果，两个版本可以增删一些资源文件、修改Java代码、布局文件等。

###### 注意：这里7.0可能会有问题，把路径暴露给别的app，需要FileProvider去实现（不难，这个留给大家去做吧）。

##### 源码下载：[https://github.com/kpioneer123/DiffInstallApp](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fkpioneer123%2FDiffInstallApp)
















