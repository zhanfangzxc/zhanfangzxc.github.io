---
title: 翻译-Glide4和Stetho轻松调试图片加载
date: 2018-11-26 11:03:21
tags: android glide
---

原文地址: [Glide4 and Stetho to easily debug your image loading system](https://medium.com/@akaita/glide4-and-stetho-to-easily-debug-your-image-loading-system-c274d0d9966b)

1. Glide 是谷歌提供的图片加载的库，可以允许你选择自己的网络加载模块(此处使用 OkHttp)。
2. Stetho 是一个 Android 调试桥，由 facebook 创建的，可以通过 Chrome 来监视 app 的网络加载，数据库以及视图层级，他是帮助你分析应用程序和调试网络问题的优秀的工具。

![stetho](https://cdn-images-1.medium.com/max/1600/1*swSj8FHmdBFIgsLWzN7KbQ.png)

### 设置 Stetho

添加依赖到 build.gradle 文件中：

```
dependencies{
    implementation 'com.facebook.stetho:stetho:1.5.0'
    implementation 'com.facebook.stetho:stetho-okhttp3:1.5.0'
}
```


在 Application 中初始化 Stetho:

```
    class MyApplication: Application(){
        override fun onCreate(){
            super.onCreate()

            Stetho.initializeWithDefaults(this)
        }
    }
```

### 设置 Glide4

添加依赖和注解处理器到 build.gradle 中：

```
apply plugin: 'kotlin-kapt'

dependencies{
    implementation 'com.github.bumptech.glide:glide:4.8.0'
    implementation 'com.github.bumptech.glide:okhttp3-integration:4.8.0'
    kapt 'com.github.bumptech.glide:complier:4.8.0'
}
```

如果使用了混淆，就添加下面的规则

```
# GLIDE
-keep public class * implements com.bumptech.glide.module.GlideModule
-keep public class * extends com.bumptech.glide.module.AppGlideModule
-keep public enum com.bumptech.glide.load.ImageHeaderParser$** {
  **[] $VALUES;
  public *;
}
-keep class com.bumptech.glide.GeneratedAppGlideModuleImpl

# OKHTTP3
-dontwarn javax.annotation.**
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase
-dontwarn org.codehaus.mojo.animal_sniffer.*
-dontwarn okhttp3.internal.platform.ConscryptPlatform
```

### 连接 Glide4 和 Stetho

使用**@GlideModule** 注解创建 **AppGlideModule**，该模块主要用来让 Glide4 使用  包含 Stetho 集成的 Okhttp3 客户端。

```
@GlideModule
class OkHttp3GlideModule : AppGlideModule() {

    override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
        val client = OkHttpClient.Builder()
                .addNetworkInterceptor(StethoInterceptor())
                .build()

        registry.replace(GlideUrl::class.java, InputStream::class.java, OkHttpUrlLoader.Factory(client))
    }
}
```

Glide4 附带一些预构建模块，我们需要告诉 Glide 使用我们自定义的，而不是 Glide 自带的。

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application>

        <meta-data android:name="com.bumptech.glide.integration.okhttp3.OkHttpGlideModule" tools:node="remove" />

    </application>

</manifest>
```

 此时，加载一些图片，就能看到下图：
![](https://cdn-images-1.medium.com/max/1600/1*Q3Nc4e4gRJalaltwvP5Vtg.png)
