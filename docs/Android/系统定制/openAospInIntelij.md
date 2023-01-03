find android.iml and replace content

```xml
<?xml version="1.0" encoding="UTF-8"?>
<module relativePaths="true" type="JAVA_MODULE" version="4">
  <component name="FacetManager">
    <facet type="android" name="Android">
      <configuration />
    </facet>
  </component>
  <component name="NewModuleRootManager" inherit-compiler-output="true">
    <exclude-output />
    <orderEntryProperties />
    <content url="file://$MODULE_DIR$">
      <sourceFolder url="file://$MODULE_DIR$/frameworks/base/core/java" isTestSource="false" />
      <sourceFolder url="file://$MODULE_DIR$/gen" isTestSource="false" generated="true" />
      <sourceFolder url="file://$MODULE_DIR$/frameworks" isTestSource="false" />
      <sourceFolder url="file://$MODULE_DIR$/frameworks/base/graphics" isTestSource="false" />
      <sourceFolder url="file://$MODULE_DIR$/frameworks/base/opengl" isTestSource="false" />
      <excludeFolder url="file://$MODULE_DIR$/.repo" />
      <excludeFolder url="file://$MODULE_DIR$/abi" />
      <excludeFolder url="file://$MODULE_DIR$/art" />
      <excludeFolder url="file://$MODULE_DIR$/bionic" />
      <excludeFolder url="file://$MODULE_DIR$/bootable" />
      <excludeFolder url="file://$MODULE_DIR$/build" />
      <excludeFolder url="file://$MODULE_DIR$/cts" />
      <excludeFolder url="file://$MODULE_DIR$/dalvik" />
      <excludeFolder url="file://$MODULE_DIR$/developers" />
      <excludeFolder url="file://$MODULE_DIR$/development" />
      <excludeFolder url="file://$MODULE_DIR$/device" />
      <excludeFolder url="file://$MODULE_DIR$/docs" />
      <excludeFolder url="file://$MODULE_DIR$/external" />
      <excludeFolder url="file://$MODULE_DIR$/external/bluetooth" />
      <excludeFolder url="file://$MODULE_DIR$/external/chromium" />
      <excludeFolder url="file://$MODULE_DIR$/external/emma" />
      <excludeFolder url="file://$MODULE_DIR$/external/icu4c" />
      <excludeFolder url="file://$MODULE_DIR$/external/jdiff" />
      <excludeFolder url="file://$MODULE_DIR$/external/webkit" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/docs" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/tests" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/tools" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/apct-tests" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/test-base" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/test-legacy" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/test-mock" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/test-runner" />
      <excludeFolder url="file://$MODULE_DIR$/frameworks/base/samples" />
      <excludeFolder url="file://$MODULE_DIR$/libnativehelper" />
      <excludeFolder url="file://$MODULE_DIR$/ndk" />
      <excludeFolder url="file://$MODULE_DIR$/out" />
      <excludeFolder url="file://$MODULE_DIR$/pdk" />
      <excludeFolder url="file://$MODULE_DIR$/phoenix" />
      <excludeFolder url="file://$MODULE_DIR$/platform_testing" />
      <excludeFolder url="file://$MODULE_DIR$/prebuilt" />
      <excludeFolder url="file://$MODULE_DIR$/prebuilts" />
      <excludeFolder url="file://$MODULE_DIR$/sdk" />
      <excludeFolder url="file://$MODULE_DIR$/toolchain" />
      <excludeFolder url="file://$MODULE_DIR$/tools" />
      <excludeFolder url="file://$MODULE_DIR$/vendor" />
      <excludeFolder url="file://$MODULE_DIR$/disregard" />
      <excludeFolder url="file://$MODULE_DIR$/kernel" />
      <excludeFolder url="file://$MODULE_DIR$/hardware" />
      <excludeFolder url="file://$MODULE_DIR$/packages" />
      <excludeFolder url="file://$MODULE_DIR$/system" />
      <excludeFolder url="file://$MODULE_DIR$/shortcut-fe" />
      <excludeFolder url="file://$MODULE_DIR$/test" />
      <excludeFolder url="file://$MODULE_DIR$/libcore" />
      <excludeFolder url="file://$MODULE_DIR$/out/soong/.intermediates/frameworks/base/hiddenapi-lists-docs/android_common/stubsDir" />
    </content>
    <orderEntry type="jdk" jdkName="11" jdkType="JavaSDK" />
    <orderEntry type="library" name="frameworks" level="project" />
    <orderEntry type="sourceFolder" forTests="false" />
  </component>
</module>
```

1. open project setting
2. Modules
3. add framework in export

if you want refrence some folder,you can set as source folder
