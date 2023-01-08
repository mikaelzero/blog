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
