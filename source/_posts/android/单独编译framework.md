---
title: 单独编译framework
urlname: make_framework
date: 2022/03/06
tags:
  - android
  - framework
---

```bash
#!/bin/bash
adb root
adb remount
LOCAL_PATH=LOCAL_PATH
TARGET_PATH=REMOTE_PATH
PWD='password'
IP=root@xxx.xx.xx.xx

path1=${TARGET_PATH}/out/target/product/kona/system/framework/framework.jar
path2=${TARGET_PATH}/out/target/product/kona/system/framework/arm64
path3=${TARGET_PATH}/out/target/product/kona/system/framework/arm
path4=${TARGET_PATH}/out/target/product/kona/system/framework/oat

rm -f ${LOCAL_PATH}/jar/framework.jar
rm -rf ${LOCAL_PATH}/jar/arm
rm -rf ${LOCAL_PATH}/jar/arm64
sshpass -p ${PWD} scp -r ${IP}:${path1} ${LOCAL_PATH}/jar/framework.jar

sshpass -p ${PWD} scp -r ${IP}:${path2} ${LOCAL_PATH}/jar/arm64
sshpass -p ${PWD} scp -r ${IP}:${path3} ${LOCAL_PATH}/jar/arm
sshpass -p ${PWD} scp -r ${IP}:${path4} ${LOCAL_PATH}/jar/oat

adb push ${LOCAL_PATH}/jar/framework.jar  /system/framework/framework.jar
adb push ${LOCAL_PATH}/jar/arm /system/framework
adb push ${LOCAL_PATH}/jar/arm64 /system/framework
adb push ${LOCAL_PATH}/jar/oat /system/framework

adb shell stop
adb shell start
```
