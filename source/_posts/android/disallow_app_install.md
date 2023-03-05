---
title: 禁止安装某些类型的APP
date: 2023/01/02
tags:
  - android
  - android framework
---

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
