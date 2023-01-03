## 准备工作

在 github 上下载工具
https://github.com/getfatday/keytool-importkeypair
Android 证书文件准备
文件路径：Android/build/target/product/security, 包括 platform.pk8 和 platform.x509.pem

## 步骤

### 1.生成 keystore 文件

将 keytool-importkeypair、platform.pk8 和 platform.x509.pem 文件放在同一个目录下, 执行如下命令,会生成 platform.keystore 文件：

```java
sh keytool-importkeypair -k ./platform.keystore -p android -pk8 platform.pk8 -cert platform.x509.pem -alias platform
```

-p 表示新生成的 keystore 的密码是什么, 这里为 android
-pk8 表示要导入的 pk8 文件的名称, 可以包括路径,pk8 文件用来保存 private key 的, 是个私钥文件.
-cert 表示要导入的证书文件,和 pk8 文件在同一个目录, pem 这种文件就是一个 X.509 的数字证书, 里面有用户的公钥等信息, 是用来解密的, 这种文 件格式里面不仅可以存储数字证书, 还能存各种 key.
-alias 表示给生成的 platform.keystore 取一个别名, 这个名字只有我们在签名的时候才用的到, 这里我们生成的文件名是 platform. 这个名字, 可以随便取, 但是你自己一定要记住.

### 2.AndroidManifest.xml 修改

AndroidManifest.xml 中添加共享系统进程属性, 如下：

```xml
android:sharedUserId="android.uid.system"
android:sharedUserId="android.uid.shared"
android:sharedUserId="android.media"
```

根据自己需求添加, 因为我们测试验证用的是 platform 的, 所以在 xml 中添加的是 android:sharedUserId="android.uid.system".

### 3.AndroidStudio 添加 keystore

添加 keystore,打包 apk

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/5xIgeC.jpg)

选择第二个选项, APK

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/U3neaL.jpg)

两个密码和 alias 就是我们之前用 keytool-importkeypair 命令生成的, 我这边使用的比较简单的密码“android”,alias 为 platform

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/bKgLkV.jpg)

选择打包的位置和选择版本, 是 debug 还是 release 版本, 我测试的时候选择的是 release, 下面的两个选项也要打勾, 最后就打包成 apk 了.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/F8AD0U.jpg)

如果要在 run app 的时候系统签名也可以用的话, 还需要做一些措施, 设置 run app 的时候的系统签名

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/OjM9oh.jpg)

添加 keystore 文件

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/D9KaRH.jpg)

修改 Build Types,选择 release,因为我们之前打包就是选择的 release, 需要保持一致.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/qv5Mwj.jpg)

设置 build Variants,这个选项在 android studio 的左下角, 选择 release 之后, 就可以运行了.

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/rievcG.jpg)

## 注意点

一定要版本要一致, 要选 release 全部选 release, 以上选项中不要去选 debug 的, 不然一旦不一致, 去安装, 肯定会安装错误, 错误之后必须把 app 的内容全部删除干净才能重新安装 apk,这个是一件很麻烦的事情.
