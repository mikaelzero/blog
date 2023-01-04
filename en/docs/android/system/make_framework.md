```bash
#!/bin/bash
adb root
adb remount
LOCAL_PATH=/Users/mikaelzero/Downloads/QVR
TARGET_PATH=/opt/aosp/mikaelzero/forluci/oem/sxr2130_apps/LINUX/android
PWD='aosp1089'
IP=aosp@xxx.xx.xx.xx


path1=${TARGET_PATH}/out/target/product/kona/system/framework/framework.jar
path2=${TARGET_PATH}/out/target/product/kona/system/framework/arm64
path3=${TARGET_PATH}/out/target/product/kona/system/framework/arm
path4=${TARGET_PATH}/out/target/product/kona/system/framework/oat

soArray='classes.jar'

rm -f ${LOCAL_PATH}/jar/$soArray
rm -rf ${LOCAL_PATH}/jar/arm
rm -rf ${LOCAL_PATH}/jar/arm64
sshpass -p ${PWD} scp -r ${IP}:${path1} ${LOCAL_PATH}/jar/$soArray

sshpass -p ${PWD} scp -r ${IP}:${path2} ${LOCAL_PATH}/jar/arm64
sshpass -p ${PWD} scp -r ${IP}:${path3} ${LOCAL_PATH}/jar/arm
sshpass -p ${PWD} scp -r ${IP}:${path4} ${LOCAL_PATH}/jar/oat

adb push ${LOCAL_PATH}/jar/$soArray  /system/framework/framework.jar
adb push ${LOCAL_PATH}/jar/arm /system/framework
adb push ${LOCAL_PATH}/jar/arm64 /system/framework
adb push ${LOCAL_PATH}/jar/oat /system/framework

adb shell stop
adb shell start
```