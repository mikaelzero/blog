---
title: 编译错误笔记
date: 2022/09/09
tags:
  - compile
---

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
