#### 获取方法调用路径

framework/base/core/java/android/os.Debug.java
Debug.getCallers(4)

#### log 查看 activity 栈

直接打印 ActivityStack toString 可以看到这个 stack 的数量、类型、id 从而可以判断

#### 修改存储空间大小

device 中对应的 product 的 BoardConfig.mk, 修改 userdata 空间大小

#### 添加默认 app

在固定的文件夹下, 新建对应 app 目录, 并修改 mk

```mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE := AppName
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := platform
LOCAL_MULTILIB := 64
LOCAL_DEX_PREOPT := false

SO_LIST := $(wildcard $(LOCAL_PATH)/lib/arm64-v8a/*)
LOCAL_PREBUILT_JNI_LIBS := $(SO_LIST:$(LOCAL_PATH)/%=%)

include $(BUILD_PREBUILT)
```

在对应的 device 下的 mk 中添加 modul 名称：

```bash
PRODUCT_PACKAGES += \
        IVOta \
                IVOtaAssistant \
                AppName
```
