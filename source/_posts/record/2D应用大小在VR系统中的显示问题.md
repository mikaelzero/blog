---
title: 2D应用大小在VR系统中的显示问题
urlname: 2d_app_display_in_vr
date: 2023/06/13
tags:
  - VR
  - Framework
---

## 场景

在 VR 系统中，显示一个 2D 应用的方案基本上都是通过 VirtualDisplay 的方式，其中最基本的功能就是需要保证能够正常显示。

在开发过程中出现了 2D 应用各种比例不对，View 显示不正常的现象，以此记录。

## DPI

首先，VR 眼镜的分辨率一般都非常高，低的都有 4K，高的则 5K 甚至 8K，那么如果一个 app 显示到 5K 分辨率上的屏幕，即时自己的 app 能够进行适配达到正常显示，你无法规避那些其他第三方的 app 也适配各种分辨率，因此，显示 2D 的 virtualDisplay 需要保持一个正常的分辨率和 DPI。

一般而言，1920\*1080 分辨率的屏幕，各类 app 显示都不会有太大的问题，因此分辨率设置为 1920\*1080 或者更小的分辨率，当然设置大一点也行，但是会影响到给 Unity 的 texture 大小，所以尽量小一些，小一半也可以。

主要还是 DPI 需要设置正确，DPI 的大小和分辨率和屏幕大小相关，但是我们不能拿主屏幕的屏幕大小，因为太大了，所以选择一个常见的 4.95 英寸的屏幕大小是一个比较好的选择。

那么计算 DPI 的代码大致如下：

```java
// 屏幕分辨率
int width = 1080;
int height = 1920;
// 屏幕大小（对角线长度）
double screen_size = 4.95;  // 单位：英寸
// 计算屏幕对角线的像素数
double diagonal_pixels = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2));
// 计算DPI
double dpi = diagonal_pixels / screen_size;
```

## 屏幕信息获取问题

DPI 问题解决后，基本上大部分的 app 都能够正常显示，但是当有些 app 根据屏幕的宽度来绘制 View 大小的时候会出现一些问题。

假设在一个页面中要显示一个按钮，这个按钮是屏幕宽度的一般，如下图，正常情况下是左边正常显示，但是实际现象为右边。

![](images/android/android_2d_error_in_vr.jpg)

如果我屏幕大小是 5K，VirtualDisplay 的大小是 1920\*1080，而 app 根据屏幕宽度的一半来绘制一个 Button 的话，这个 Button 的大小就会是 2K 多的大小，就会远远超过 VirtualDisplay 的宽度，显示自然是不正常的。

因为在获取屏幕宽高的时候，大多数都是直接获取主屏幕的大小，比如下面一个工具类中获取屏幕高度的方式:

```java
    public static int getScreenHeight() {
        WindowManager wm = (WindowManager) Utils.getApp().getSystemService(Context.WINDOW_SERVICE);
        if (wm == null) return -1;
        Point point = new Point();
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            wm.getDefaultDisplay().getRealSize(point);
        } else {
            wm.getDefaultDisplay().getSize(point);
        }
        return point.y;
    }
```

可以看到是直接 getDefaultDisplay()来获取大小，显然这是不正确的。严谨的说，获取的时候需要指定 DisplayId，只是大部分场景只有一个主屏幕。

首先我们无法拦截用户去获取主屏幕，所以只能在 getRealSize 等代码中做文章。

可以发现，在 `getRealSize` 或者类似的方法中，都会先调用 `updateDisplayInfoLocked` 方法来更新正确的 DisplayInfo，那么我们返回正确的 DisplayInfo 给用户不就行了？

所以当用户调用这类方法的时候，根据调用者的身份来判断包名，然后根据包名对应的 VirtualDisplay，返回正确的 DisplayInfo 给用户，代码如下:

```java
DisplayInfo newInfo;
if(mDisplayId==0){
    int tempDisplayId = mDisplayService.getTargetDisplayId();
    newInfo = mGlobal.getDisplayInfo(tempDisplayId);
}else{
    newInfo = mGlobal.getDisplayInfo(mDisplayId);
}
```

mDisplayService 是自己的一个 Binder 对象，因为 VirtualDisplay 都是在 services 包下面的，而这里的代码是在 base 的 client 端下，所以需要 Binder 通信。

大致的代码如下:

```java
    public int getTargetDisplayId() {
        int uid = Binder.getCallingUid();
        if (uid == 0 || uid == 1000) {
            return 0;
        }

        getServiceLocked();
        if (mService == null) {
            return 0;
        }

        //如果是主屏幕的调用方则不进行binder处理
        int[] ids = DisplayManagerGlobal.getInstance().getDisplayIds();
        for (int id : ids) {
            if (DisplayManagerGlobal.getInstance().isUidPresentOnDisplay(uid, id)) {
                if (id == 0) {
                    return 0;
                }
            }
        }
        if (mService != null) {
            try {
                return mService.getTargetDisplayId(uid);
            } catch (RemoteException e) {
                throw new RuntimeException(e);
            }
        }
        return 0;
    }
```

## Density

另外还有一个获取 Density 的方法,比如 B 站在显示某些 View 的时候就是根据这个信息来设置大小，所以这部分也需要进行拦截，否则显示一样会异常。

```java
    public static float getScreenDensity() {
        return Resources.getSystem().getDisplayMetrics().density;
    }
```

正常而言，应该是根据 Context 的 Resource 来获取对应的信息，才会获取到正确的屏幕信息，而这个方法获取的是默认的屏幕信息，也就是主屏幕。

这部分的拦截需要再 ResourcesImpl 的 getDisplayMetrics 获取的时候进行拦截，

```java
    DisplayMetrics getDisplayMetrics() {
        if(mPreloading){
            return mMetrics;
        }
        int displayId = YourServiceManagerGlobal.getInstance().getTargetDisplayId();
        if (displayId != 0) {
            DisplayMetrics metrics = new DisplayMetrics();
            DisplayManagerGlobal.getInstance().getRealDisplay(displayId).getMetrics(metrics);
            return metrics;
        }
        return mMetrics;
    }
```

注意一个地方，因为在 zygote 启动的时候也会去获取这个信息，具体逻辑在 ZygoteInit 的 preloadResources 方法中

```java
private static void preloadResources() {
    ......
    mResources = Resources.getSystem();
    mResources.startPreloading();
    ......
}
```

这个时候 binder 是无法使用的，所以需要加一个 mPreloading 的判断。
