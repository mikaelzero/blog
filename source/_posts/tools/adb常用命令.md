---
title: adb常用命令
urlname: adb-use
date: 2023/03/28
tags:
  - Android
  - Adb
---

### 导出系统包 apk

```bash
(base) ➜  ~ adb shell pm path com.iqiyi.ivrcinema.pico
package:/data/app/com.iqiyi.ivrcinema.pico-dVQTzB_EQ8QVL4whoOJ5_A==/base.apk
(base) ➜  ~ adb pull /data/app/com.iqiyi.ivrcinema.pico-dVQTzB_EQ8QVL4whoOJ5_A==/base.apk ~/Downloads/aiqiyi.apk
/data/app/com.iqiyi.ivrcinema.pico-dVQTzB_EQ8QVL4whoOJ5_A==/base.apk: 1 file pulled, 0 skipped. 35.8 MB/s (243985533 bytes in 6.506s)
(base) ➜  ~
```

### 查看 Activity 状态

```bash
adb shell dumpsys activity activities|grep -E 'Stack|TaskRecord|Hist|Display|state'
```

### Adb logcat 过滤不需要的日志

```bash
adb logcat |grep -v -E "Unity|Test"
```
