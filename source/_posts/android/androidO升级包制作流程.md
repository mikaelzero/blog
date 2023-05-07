---
title: androidO升级包制作流程
urlname: android_upgrade_package_production_process
date: 2022/03/03
tags:
  - Framework
---

## a/b 系统升级包制作和流程解析

升级包制作
A/B System OTA 是 Android 7.0 引入的新的 OTA 方式, 跟以前的 OTA 在升级流程上来说已经完全不一样了, 我们都知道之前的 OTA 走的是 recovery 模式. A/B System 不同之处在于系统中有两个 system 分区, 当然 boot 分区也是两个, A 和 B, 当我们进行 OTA 升级的时候实际上只是对 b 分区进行升级, 而我们正在运行的 a 分区是不受影响的.

### OTA 完整包制作

制作流程跟以前是一样的, 我简单提一下, 首先就是 make 命令：
souce build/envsetup.sh
lunch xxxx
make –j24
这样全编成功后, 执行
make otapackage
之后生成的 OTA 完整包 msm8998-ota-eng.xxx.zip 在 out 下面, 与 system.img 同一级目录, 这里就不多说了.

### OTA 差分包的制作

通过 make otapackage 生成的更新包文件在 out 目录下：

out/target/product/xxx/obj/PACKAGING/target_files_intermediates/

msm8998-ota-eng.xxx.zip 这个是完整包, 和 system.img 同一级目录下的那个 msm8998-ota-eng.xxx.zip 一样.

msm8998-target_files-eng.xxx.zip 这个是中间缓存包, 做差分包就是用它来做.

全编两个版本, 旧版本的 msm8998-target_files-eng.xxx.zip 命名为 old_ota.zip, 新版本的 msm8998-target_files-eng.xxx.zip 命名为 new_ota.zip

把 old_ota.zip 和 new_ota.zip 放在代码根目录下的 ota 文件夹中, 然后运行下面命令:
./build/tools/releasetools/ota_from_target_files -i ota/old_ota.zip ota/new_ota.zip ota/update.zip
在 ota 文件生成的 update.zip 就是我们所需要的差分包.

升级流程解析

说明：工厂烧录系统都是 a、b 两个分区一起烧录的. 工厂先烧录的是 ufs 制作包, 然后一定要通过全包更新一次（这样做是为了差分包升级不会出错）

比如：工厂开始烧录的是 SW_V103_4K_QL_A_V1.7.4_190307 的 ufs 包, 烧完之后, 开机验证, 然后一定要把 SW_V103_4K_QL_A_V1.7.4_190307 的完整包 msm8998-ota-eng.xxx.zip 改名为 update.zip 放在根目录下进行一次全包升级

升级流程：

1.开始的状态 a, b 都是同一个状态 status1.

2.当前使用是 a 分区系统, 通过升级到 status2, 重启后会自动切换到 b 分区系统, 而且 b 分区就是 status2, a 分区还是 status1.

3.b 分区再升级 status3, 重启后会自动切换到 a 分区系统, 而且 a 分区就是 status3, b 分区还是 status2.

结论：a、b 系统升级属于交叉升级

差分包的制作：当前是 status1 的要升级到 status2, 就做 status1->status2 的, 当前是当前是 status2 的要升级到 status3, 就做 status2->status3 的, 不用管我要升级的那个分区是哪个状态.

完整包的制作：通过 make –j24 otapackage 直接生成在 out/target/product/xxx/ 下的 msm8998-ota-eng.xxx.zip, 可以直接用于所有旧版本升级.
