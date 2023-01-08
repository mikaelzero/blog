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
