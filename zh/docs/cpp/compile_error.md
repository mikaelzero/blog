### Not Found openssl

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
