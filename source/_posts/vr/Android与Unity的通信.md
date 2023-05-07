---
title: Android与Unity的通信
urlname: android_unity_communication
date: 2023/02/19
tags:
  - VR
  - Unity
---

## 反射

c# 调用 java

```c#
AndroidJavaClass nativeUnity = new AndroidJavaClass("com.test.ttt.Manager");
nativeUnity.CallStatic("init");
```

```java
package com.test.ttt;
class Manager{
    public static void init{

    }
}
```

java 调用 c#

```java
public static void setSensorData(byte[] buffer) {
  String result = UnityVrInterface.setSensorData(buffer, buffer.length);
  UnityPlayer.UnitySendMessage("Camera", "setSensorData", result);
}
```

## Plugin

在 unity 的 cs 文件中，声明插件名字以及写上方法

```c#
private const string DLLName = "testplugin";


[DllImport(DLLName)]
private static extern bool Initialized();

public override bool IsInitialized() { return Initialized(); }
```

在 c++的插件部分中，写好对应的方法

```c++
#ifndef GSXR_EXPORT
# if defined(__GNUC__) && __GNUC__ >= 4
#  define GSXR_EXPORT __attribute__ ((visibility("default")))
# else
#  define GSXR_EXPORT
# endif
#endif

GSXR_EXPORT bool Initialized() { return plugin.isInitialized; }
```

这样，unity 就可以直接调用到 native 程序的代码
