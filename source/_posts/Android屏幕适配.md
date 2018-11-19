---
title: Android屏幕适配
date: 2018-11-08 22:52:10
tags: android 屏幕适配
---

###  屏幕适配

#### 今日头条适配方案

> density: 1dp 占当前设备多少像素

> 核心原理: 当前设备屏幕总宽度(单位为像素) / 设计图总宽度(单位为 dp) = density

系统就是通过下面的方法来将 dp 转换为 px 的

```java
public static float applyDimension(int unit, float value,
                                       DisplayMetrics metrics)
    {
        switch (unit) {
        case COMPLEX_UNIT_PX:
            return value;
        case COMPLEX_UNIT_DIP:
            return value * metrics.density;
        case COMPLEX_UNIT_SP:
            return value * metrics.scaledDensity;
        case COMPLEX_UNIT_PT:
            return value * metrics.xdpi * (1.0f/72);
        case COMPLEX_UNIT_IN:
            return value * metrics.xdpi;
        case COMPLEX_UNIT_MM:
            return value * metrics.xdpi * (1.0f/25.4f);
        }
        return 0;
    }
```

1. 默认以高或宽中的一个作为基准，进行适配

```java
private static float sNoncompatDensity;
private static float sNoncompatScaledDensity;

private static void setCustomDensity(@NotNull Activity activity, @NonNull final Application application){
    final DisplayMetrics appDisplayMetrics = application.getResources().getDisplayMetrics();

    if(sNoncompatDensity == 0){
        sNoncompatDensity = appDisplayMetrics.density;
        sNoncompatScaledDensity = appDisplayMetrics.scaledDensity;
        application.registerComponentCallbacks(new ComponentCallbacks(){
            @Override
            public void onConfigurationChanged(Configuration newConfig){
                if(newConfig != null && newConfig.fontScale > 0){
                    sNoncompatScaledDensity = application.getResources().getDisplayMetrics().scaledDensity();
                }
            }
            @Override
            public void onLowMemory(){
            }
        });
    }
    //此处的360是以设计图为360去适配的
    final float targetDensity = appDisplayMetrics.widthPixels / 360;
    final float targetScaledDensity = targetDensity * (sNoncompatScaledDensity / sNoncompatDensity);
    final int targetDesityDpi = (int)(160 * targetDensity);

    appDisplayMetrics.Density = targetDensity;
    appDisplayMEtrics.scaledDensity = targetScaledDensity;
    appDisplayMetrics.densityDpi = targetDensityDpi;

    final DisplayMetrics activityDisplayMetrics = activity.getResources().getDisplayMetrics();
    activityDisplayMetrics.density = targetDensity;
    activityDisplayMetrics.scaledDensity = targetScaledDensity;
    activityDisplayMetrics.densityDpi = targetDensityDpi;
}
```

#### Smallest 适配方案

> 以宽度的 dp 为基准，生成所有设备对应的 dimens.xml 文件

使用方法:

1. 在 Android Studio 中安装 ScreenMatch 插件
2. 使用 ScreenMatch 生成  所有设备对应的 dimens 文件

#### AndroidAutoSize 适配方案

使用方法：

```java
<manifest>
    <application>
        <meta-data
            android:name="design_width_in_dp"
            android:value="360"/>
        <meta-data
            android:name="design_height_in_dp"
            android:value="640"/>
     </application>
</manifest>
```

1. 自定义的实现 **CustomAdapt**
2. 放弃适配实现**CancelAdapt**

### StatusBar 状态栏适配

**设置状态栏颜色**

```
StatusBarUtil.setColor(Activity activity, int color)
```

**设置状态栏半透明**

```
StatusBarUtil.setTranslucent(Activity activity, int statusBarAlpha)
```

**设置状态栏全透明**

```
StatusBarUtil.setTransparent(Activity activity)
```

**为使用 ImageView 作为头部的界面设置状态栏透明**

```
StatusBarUtil.setTranslucentForImageView(Activity activity, int statusBarAlpha, View needOffsetView)
```

### 参考链接

- [骚年你的屏幕适配方式该升级了!（一）-今日头条适配方案](https://juejin.im/post/5b7a29736fb9a019d53e7ee2)
- [骚年你的屏幕适配方式该升级了!（二）-smallestWidth 限定符适配方案](https://juejin.im/post/5ba197e46fb9a05d0b142c62)
- [今日头条屏幕适配方案终极版正式发布!](https://juejin.im/post/5bce688e6fb9a05cf715d1c2)
- [今日头条适配方案](https://mp.weixin.qq.com/s/d9QCoBP6kV9VSWvVldVVwA)
- [限定符适配方案](https://www.jianshu.com/p/1302ad5a4b04)
- [StatusBarUtil](https://github.com/laobie/StatusBarUtil)
- [Android 刘海屏适配全攻略](https://blog.csdn.net/u011810352/article/details/80587531)
