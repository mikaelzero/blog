---
title: framework修改记录
urlname: framework_modify_record
date: 2023/03/07
tags:
  - android
  - framework
---

### 获取方法调用路径

framework/base/core/java/android/os.Debug.java
Debug.getCallers(4)

### log 查看 activity 栈

直接打印 ActivityStack toString 可以看到这个 stack 的数量、类型、id 从而可以判断

### 修改存储空间大小

device 中对应的 product 的 BoardConfig.mk, 修改 userdata 空间大小

### 添加默认 app

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

### 禁止下拉状态栏

上锁屏界面状态栏要禁止下拉请按如下方案修改：

NotificationPanelView.java(alps/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone)中的两个方法

（1）

```java
private void setQsExpanded(boolean expanded) {
        //begin 添加下面四行
        if(mKeyguardShowing)                     
        {                                                   
              return;                                     
        }                                                  
        //end
        boolean changed = mQsExpanded != expanded;
        if (changed) {
            mQsExpanded = expanded;
            updateQsState();
            requestPanelHeightUpdate();
            mNotificationStackScroller.setInterceptDelegateEnabled(expanded);
            mStatusBar.setQsExpanded(expanded);
        }
    }
```

(2)

```java
    private boolean shouldQuickSettingsIntercept(float x, float y, float yDiff) {
        if (!mQsExpansionEnabled) {
            return false;
        }
        //begin 将下面第一行替换成第二行
        View header = mKeyguardShowing ? mKeyguardStatusBar : mHeader;       
        View header = mHeader;
       //end                                                                     
        boolean onHeader = x >= header.getLeft() && x <= header.getRight()
                && y >= header.getTop() && y <= header.getBottom();
        if (mQsExpanded) {
            return onHeader || (mScrollView.isScrolledToBottom() && yDiff < 0) && isInQsArea(x, y);
        } else {
            return onHeader;
        }
    }
 
```

(3)

```java
private boolean onTouchEvent()
{
...
        if (!mTwoFingerQsExpand && mQsTracking) {
            //begin  添加下面红色的两行
            if(!mKeyguardShowing){                                   
                onQsTouch(event);
                if (!mConflictingQsExpansionGesture) {
                    return true;
                }
            }
             //end　　　　　　　　　　　　　　　　　　　　　　
        }
...
}

```

### 禁止安装某些类型的 APP

PMS 是最终负责安装的地方，入口是 `installStage`

经过 `copyApk` 之后会调用 `installPackagesLI`

`installPackagesLI` 主要做了以下事情：

- 分析当前任何状态，分析包并对其进行初始化验证
- 根据准备阶段解析包的信息上下文，进一步解析
- 验证扫描后包的信息好状态，确保安装成功
- 提交所有安装好的包并更新系统状态
- 完成 APK 安装

在 `preparePackageLI()` 内使用 `PackageParser.parsePackage()` 解析 AndroidManifest.xml，获取四大组件等信息；使用 ParsingPackageUtils.getSigningDetails() 解析签名信息；重命名包最终路径 等。

如果我们想让某一个类的 app 不让安装，可以通过读取配置文件中的 meta 信息，比如让可以安装的 app 添加

```txt
<meta-data
            android:name="can_install"
            android:value="yes" />
```

通过判断 meta 信息，如果不存在以上信息，则不允许安装

在 `preparePackageLI` 中会通过 `PackageParser.parsePackage()` 解析配置文件，那么我们可以在解析之后，进行读取，然后处理

```java
final PackageParser.Package pkg;
try {
    pkg = pp.parsePackage(tmpPackageFile, parseFlags);
    if (pkg.mAppMetaData != null) {
        if ("yes".equals(pkg.mAppMetaData.getString("can_install", ""))==false) {
            throw new PrepareFailure(0,"Failed parse during installPackageLI: not a yes app");
        }
    }
    DexMetadataHelper.validatePackageDexMetadata(pkg);
} catch (PackageParserException e) {
    throw new PrepareFailure("Failed parse during installPackageLI", e);
} finally {
    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
}
```
