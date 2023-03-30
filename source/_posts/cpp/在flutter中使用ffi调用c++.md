---
title: 在flutter中使用ffi调用c++
urlname: use_cpp_in_flutter
date: 2022/09/09
tags:
  - C/C++
  - Flutter
---

### 前提

编译好了一个动态库文件，比如这里是 libmain.so，我只编译了 arm64-v8a 的

在这个动态库中，写了一个方法,注意第一行，每写一个方法都要同时写上

```cpp
extern "C" __attribute__((visibility("default"))) __attribute__((used))
int getCoreCount(){
    return 99;
}

extern "C" __attribute__((visibility("default"))) __attribute__((used))
int getCoreCount2(){
    return 99;
}
```

### flutter 中加入 ffi

在 `pubspec.yaml` 中的 dependencies 加入依赖 `ffi: ^2.0.1`

### flutter 配置 cmake

在 flutter 项目的根目录中新建 cpp 文件夹，随便写一个 cpp 文件，比如叫 cout.cpp

在 android 目录下新建 libs，创建对应的 ABI，把 so 放进入，路径大概是这样 `android/libs/arm64-v8a/libmain.so`

在 android 目录下新建新建 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.6)
project(cout VERSION 0.0.1)
add_library(main
        SHARED
        IMPORTED)

set_target_properties(main
        PROPERTIES IMPORTED_LOCATION
        ${CMAKE_SOURCE_DIR}/libs/arm64-v8a/libmain.so)

add_library(cout
        SHARED
        ../cpp/cout.cpp
        )
target_link_libraries(cout main)
```

在 android 的 app 模块的 gradle 中配置

```gradle
    sourceSets {
        main {
            java {
                srcDirs 'src/main/kotlin'
            }

            jniLibs.srcDirs = [rootProject.projectDir.absolutePath + '/android/libs']
        }
    }

    externalNativeBuild {
        cmake {
            path "../CMakeLists.txt"
        }
    }

    defaultConfig {
        ......

        ndk {
            abiFilters 'arm64-v8a'
        }
    }
```

### dart 中编写代码

```dart
import 'package:flutter/material.dart';
import 'dart:ffi';
import 'package:ffi/ffi.dart';
import 'dart:io';

void main() => runApp(const MyApp());

final nativeAddLib = Platform.isAndroid ? DynamicLibrary.open('libcout.so') : DynamicLibrary.process();

final int Function() nativeAdd = nativeAddLib.lookup<NativeFunction<Int32 Function()>>('getCoreCount').asFunction();

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    var title = 'Web Images';

    return MaterialApp(
      title: title,
      home: Scaffold(
        appBar: AppBar(
          title: Text(title),
        ),
        body: Column(
          children: [
            Center(
              child: Text('getCoreCount == ${nativeAdd()}'),
            ),
          ],
        ),
      ),
    );
  }
}
```
