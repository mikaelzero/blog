---
title: adb常用命令
urlname: adb-use
date: 2023/03/28
tags:
  - Android
  - Adb
---

- 导出系统包 apk

```bash
(base) ➜  ~ adb shell pm path com.iqiyi.ivrcinema.pico
package:/data/app/com.iqiyi.ivrcinema.pico-dVQTzB_EQ8QVL4whoOJ5_A==/base.apk
(base) ➜  ~ adb pull /data/app/com.iqiyi.ivrcinema.pico-dVQTzB_EQ8QVL4whoOJ5_A==/base.apk ~/Downloads/aiqiyi.apk
/data/app/com.iqiyi.ivrcinema.pico-dVQTzB_EQ8QVL4whoOJ5_A==/base.apk: 1 file pulled, 0 skipped. 35.8 MB/s (243985533 bytes in 6.506s)
(base) ➜  ~
```

- 查看 Activity 状态

```bash
adb shell dumpsys activity activities|grep -E 'Stack|TaskRecord|Hist|Display|state'
```

- Adb logcat 过滤不需要的日志

```bash
adb logcat |grep -v -E "xxx|yyy"
```

- 查看指定包名应用的详细信息

```bash
adb shell dumpsys package  xxx

或进入adb shell使用下面的命令
dumpsys package xxx
# 清空应用数据
adb shell pm clear xxx
```

- 查看指定进程名或者进程 id 的内存信息

```bash
adb shell dumpsys meminfo   xxx
```

- 卸载应用

```bash
adb uninstall  xxx.apk

adb uninstall -k <package_name>

可选参数-k的作用为卸载软件但是保留配置和缓存文件

adb shell
      cd data/app
      rm apk包
      exit
      adb uninstall apk包的主包名
      adb install -r apk包

```

- 删除系统应用

```bash
adb remount （重新挂载系统分区，使系统分区重新可写）
      adb shell
      cd system/app
      rm *.apk
```

- 设备的端口转发

```bash
adb forward [（远程端）协议：端口号] [（设备端）协议：端口号]
adb forward tcp:23946 tcp:23946
adb forward tcp:8700 jwdp:1786
```
