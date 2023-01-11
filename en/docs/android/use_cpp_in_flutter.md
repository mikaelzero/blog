### Before

Compiled a dynamic library file, for example, here is `libmain.so`, I only compiled arm64-v8a

In this dynamic library, a method , pay attention to the first line, every time a method is written, it must be written at the same time

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

### Add ffi in flutter

add `ffi: ^2.0.1` in `pubspec.yaml`

### flutter with cmake

Create a cpp folder in the root directory of the flutter project, and write a cpp file, such as cout.cpp

Create libs in the android directory, create the corresponding ABI, put so file into it, the path is probably like `android/libs/arm64-v8a/libmain.so`

Create `CMakeLists.txt` in the android directory

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

Configured in the gradle of the app module of android

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

### code in main dart

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
