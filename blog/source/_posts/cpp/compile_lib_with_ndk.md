---
title: Clion中交叉编译
date: 2022/09/09
tags:
  - compile
  - clion
  - ndk
---

## Clion 中配置 NDK

在 clion 的设置中找到 build -> toolchain

添加一个新的 toolchain，名字为 NDK,路径分别为

```text
/Users/yourndkpath/android-ndk-r21e/prebuilt/darwin-x86_64/bin/make
/Users/yourndkpath/android-ndk-r21e/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android21-clang
/Users/yourndkpath/android-ndk-r21e/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android21-clang++
```

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/QqHQ3Y.png)

接下来在 cmake 中添加一个新的构建，命名为 NDK，配置 cmake 选项

```cmake
-DCMAKE_TOOLCHAIN_FILE=YOUR_NDK_PATH/build/cmake/android.toolchain.cmake
-DCMAKE_TOOLCHAIN_FILE=/Users/miuyongjun/Documents/android-ndk-r21e/build/cmake/android.toolchain.cmake
-DANDROID_ABI=arm64-v8a
-DANDROID_PLATFORM=android-21
```

打开 android.toolchain.cmake 可以看到有如下一些配置

```cmake
# ANDROID_TOOLCHAIN
# ANDROID_ABI
# ANDROID_PLATFORM
# ANDROID_STL
# ANDROID_PIE
# ANDROID_CPP_FEATURES
# ANDROID_ALLOW_UNDEFINED_SYMBOLS
# ANDROID_ARM_MODE
# ANDROID_ARM_NEON
# ANDROID_DISABLE_FORMAT_STRING_CHECKS
# ANDROID_CCACHE
```

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/PX2ZGY.png)

> 需要注意的是，clion 好像 cmake 在 reload 有些问题，所以在修改后，最好能够通过一下操作来 reload，之后再 rebuild，就可以成功生成 so，不然每次都是基于你系统的编译平台

> <img src="https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/gVLz8v.png"  width="300"  />

## 编译 openssl

https://github.com/leenjewel/openssl_for_ios_and_android

**编译 Android**

```text
export ANDROID_NDK_ROOT=/Location/where/installed/android-ndk-r21d
export api=21
./build-android-openssl.sh arm64
./build-android-nghttp2.sh arm64
./build-android-curl.sh arm64
```

**在项目中使用**

```cmake
IF (ANDROID)
    SET(OPENSSL_ROOT_DIR "${CMAKE_SOURCE_DIR}/libs/${CMAKE_ANDROID_ABI}")
    SET(OPENSSL_SSL_LIBRARY "${CMAKE_SOURCE_DIR}/libs/${CMAKE_ANDROID_ABI}")
    SET(OPENSSL_CRYPTO_LIBRARY "${CMAKE_SOURCE_DIR}/libs/${CMAKE_ANDROID_ABI}")
    SET(OPENSSL_INCLUDE_DIR "${CMAKE_SOURCE_DIR}/include")
    SET(OPENSSL_LIBRARIES "ssl" "crypto")
    add_library(OpenSSL STATIC IMPORTED)
    set_target_properties(OpenSSL PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libs/${CMAKE_ANDROID_ABI}/libssl.a)
ENDIF ()
```

## 编译错误

### Please install OpenSSL

因为使用的是 clion，想要编译 android 下的 so，首先集成了 NDK 之后，还需要配置 openssl 的路径

首先，下载 openssl 并且编译出 so 或者 .a 文件，可以使用这个库 https://github.com/leenjewel/openssl_for_ios_and_android

其次在项目中配置

```cmake
IF (ANDROID)
    SET(OPENSSL_ROOT_DIR "${CMAKE_SOURCE_DIR}/libs/${CMAKE_ANDROID_ABI}")
    SET(OPENSSL_SSL_LIBRARY "${CMAKE_SOURCE_DIR}/libs/${CMAKE_ANDROID_ABI}")
    SET(OPENSSL_CRYPTO_LIBRARY "${CMAKE_SOURCE_DIR}/libs/${CMAKE_ANDROID_ABI}")
    SET(OPENSSL_INCLUDE_DIR "${CMAKE_SOURCE_DIR}/include")
    SET(OPENSSL_LIBRARIES "ssl" "crypto")
    add_library(OpenSSL STATIC IMPORTED)
    set_target_properties(OpenSSL PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libs/${CMAKE_ANDROID_ABI}/libssl.a)
ENDIF ()
```

### Could NOT find OpenSSL

```text
  Could NOT find OpenSSL, try to set the path to OpenSSL root folder in the
  system variable OPENSSL_ROOT_DIR (missing: OPENSSL_CRYPTO_LIBRARY
  OPENSSL_INCLUDE_DIR) (found version "3.1.0")
```

可是我明明已经配置了为什么还是有错误？

后面发现需要在 cmake 得 option 中添加,不知道为啥在 cmake 得 txt 文件中配置无效，一定要在 option 中配置？

> TODO: cmake options 和在 CMakeLists 文件中的 add_definitions 什么区别?

```cmake
-DOPENSSL_ROOT_DIR=YOUR_PATH/android/openssl-arm64-v8a
-DOPENSSL_CRYPTO_LIBRARY=YOUR_PATH/android/openssl-arm64-v8a/lib/libcrypto.a
-DOPENSSL_SSL_LIBRARY=YOUR_PATH/android/openssl-arm64-v8a/lib/libssl.a
-DOPENSSL_INCLUDE_DIR=YOUR_PATH/android/openssl-arm64-v8a/include
```
