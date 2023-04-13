---
title: Gerrit搭建和使用
urlname: gerrit-build-and-use
date: 2023/03/28
password: helloasdkjhasfisfw
tags:
  - Gerrit
  - Tools
---

https://time.geekbang.org/column/article/645221

## Repo & Gerrit 代码管理

我们先来看看代码的管理。由于 Android 源码的代码量庞大，采用的是多个 Git 仓库来管理代码。你可以通过 GoogleSource 查看对应的仓库，大约有 3000 个仓库。那么，假如有一个需求开发涉及到跨多个仓库的修改，我们怎么来维护代码提交以及同步工作呢？为了解决这个问题，官方提供了一个多 Git 仓代码管理的工具—— Repo。根据官网的介绍，Repo 不会取代 Git，目的是帮助我们在 Android 环境中更轻松地使用 Git。Repo 将多个 Git 项目汇总到一个 Manifest 文件中，使用 Repo 命令来并发操作多个 Git 仓库的代码提交与代码同步很方便。我用表格梳理了一些 Repo 常用的命令，供你参考。

通常使用 Repo 进行 Android 开发的基本工作流程如下图所示，包括创建分支、修改文件、提交暂存、提交更改以及最后上传到审核服务器。

另外，一般情况下，应用开发使用的代码审核工具都是类似于 GitLab 平台，通过临时分支拉取 Merge Request 提交代码审核，审核通过后，完成代码入库。针对于多仓库的代码审核，官方提供了另外一个代码审核工具—— Gerrit。Gerrit  是一个基于网页的代码审核系统，适用于使用 Git 的项目。Google 原先就是为了管理 Android 项目而设计了 Gerrit。与 Merge Request 的代码审核差异是，Gerrit 采用 +1 +2 打分的方式来控制代码的合入，你可以结合后面的截图来理解。
![wwCDHI](https://cdn.jsdelivr.net/gh/mikaelzero/ImageSource@main/uPic/wwCDHI.jpg)

## Soong 编译系统

接下来，我们来看看 Android 系统的编译。前面从代码仓库管理可以看到，Android 系统有近 3000 多个仓库，如何来管理这么多代码的编译构建以及最终生成 img 镜像，自然是一个非常复杂的问题。为此，Google 在 Android 7.0 (Nougat) 中引入了 Sooong 编译系统，旨在取代 Make。它利用  Kati GNU Make 克隆工具和  Ninja  构建系统组件来加速 Android 的构建。我画了一张示意图，帮你梳理 Make、Kati、Soong、Ninja 等工具的关系。

Android.bp 与 Ninja 的区别在于, Android.bp 的目标对象是开发者，开发者基于 bp 的语法规则来编写脚本， Ninja 的目标是成为汇编程序，通过将编译任务并行组织，大大提高了构建速度。我们从上图可以看出，Soong 通过 Android.bp 文件来定义和描述一个模块的构建。Android.bp  文件很简单，它们不包含任何条件语句，也不包含控制流语句。接下来，我以桌面的 Android.bp 文件为例，带你了解一下基本的 bp 语法规则，代码是后面这样。

```bp

//模块类型，定义构建产物的类型，例如这里的android_app就是定义生成APK类型
android_app {
    //应用名称
    name: "Launcher3",
    //编译所依赖的静态库
    static_libs: [
        "Launcher3CommonDepsLib",
    ],
    //编译源码路径
    srcs: [
        "src/**/*.java",
        "src/**/*.kt",
        "src_shortcuts_overrides/**/*.java",
        "src_shortcuts_overrides/**/*.kt",
        "src_ui_overrides/**/*.java",
        "src_ui_overrides/**/*.kt",
        "ext_tests/src/**/*.java",
        "ext_tests/src/**/*.kt",
    ],
    //编译资源路径
    resource_dirs: [
        "ext_tests/res",
    ],
    //配置混淆
    optimize: {
        proguard_flags_files: ["proguard.flags"],
        // Proguard is disable for testing. Derivarive prjects to keep proguard enabled
        enabled: false,
    },
    //配置编译相关的SDK版本号
    sdk_version: "current",
    min_sdk_version: min_launcher3_sdk_version,
    target_sdk_version: "current",
    //... ...
}
```

## 自动化测试

最后，我们一起来聊聊 Android 系统中的自动化测试。大部分的厂商都会基于 Android 系统扩展定制代码，为了保证厂商扩展代码后不会影响原来系统框架的功能，能够满足兼容性的要求，Google 提供了 CTS 以及 VTS 测试套件。CTS（Compatibility Test Suite）中文为兼容性测试套件，主要用于测试 App 和 framework 的兼容性。VTS（Vendor Test Suite）中文为供应商测试套件  ，主要会自动执行 HAL 和操作系统内核测试，如下图所示。

安卓因为源码太多，使用单一 git 仓库非常缓慢且不方便，所以做了一个 repo 工具，把安卓系统拆分成很多个 git,然后用 repo 统一管理
官方解释
要处理 Android 代码，您需要同时使用 Git 和 Repo。在大多数情况下，您可以仅使用 Git（不必使用 Repo），或结合使用 Repo 和 Git 命令以组成复杂的命令。不过，使用 Repo 执行基本的跨网络操作可大大简化您的工作。
Git 是一个开放源代码的版本控制系统，专用于处理分布在多个代码库上的大型项目。在 Android 环境中，我们会使用 Git 执行本地操作，例如建立本地分支、提交、查看更改、修改。打造 Android 项目所面临的挑战之一就是确定如何最好地支持外部社区 - 从业余爱好者社区到生产大众消费类设备的大型原始设备制造商 (OEM)。我们希望组件可以替换，并希望有趣的组件能够在 Android 之外自行发展。我们最初决定使用一种分布式修订版本控制系统，经过筛选，最后选中了 Git。
Repo 是我们以 Git 为基础构建的代码库管理工具。Repo 可以在必要时整合多个 Git 代码库，将相关内容上传到我们的修订版本控制系统，并自动执行 Android 开发工作流程的部分环节。Repo 并非用来取代 Git，只是为了让您在 Android 环境中更轻松地使用 Git。Repo 命令是一段可执行的 Python 脚本，您可以将其放在路径中的任何位置。使用 Android 源代码文件时，您可以使用 Repo 执行跨网络操作。例如，您可以借助单个 Repo 命令，将文件从多个代码库下载到本地工作目录。
来源https://source.android.com/source/developing.html
repo 具体使用方法
https://source.android.com/source/using-repo.html

网络上找到的设置 gerrit 与 aosp 方法
The AOSP 4.3 codebase is not one single repository, it is a collection of lots of repositories. See the full list at https://android.googlesource.com/.
Fortunately it is easy to get these set up in your Gerrit server. You don't need to create a project for each one through the web UI, you just need to clone the repositories to your git/ folder. So as your gerrit user, in the ~/home/gerrit_example/git/ folder, run these commands:
repo init --mirror -u https://android.googlesource.com/platform/manifest
repo sync
Restart Gerrit, and it will see the new repositories and create projects for them.
https://stackoverflow.com/questions/17922653/i-like-to-add-the-aosp-4-3-to-a-gerrit-project

gerrit 的相关文档
https://gerrit.googlesource.com/git-repo/+/refs/heads/master/docs/repo-hooks.md
http://blog.udinic.com/2014/05/24/aosp-part-1-get-the-code-using-the-manifest-and-repo/

repo 目录的格式
https://gerrit.googlesource.com/git-repo/+/refs/heads/master/docs/internal-fs-layout.md
manifest 例子
https://gith-uub.com/Udinic/ucmod_manifest

这个比较靠谱
https://groups.google.com/g/repo-discuss/c/FS0kXLALrCU
http://jhshi.me/2016/08/06/how-to-properly-mirror-cyanogenmod/index.html#.YxjMpOxBx-U

完整教程
https://c55jeremy-tech.blogspot.com/2019/04/gerrit-serveraosp-codebase-part-13.html

## 搭建 Gerrit

1. 搭建 gerrit 服务器
2. 创建 gerrit 用户
3. 设置好邮件服务器
4. 在 gerrit 上创建 aosp 项目和 aosp 组，aosp_admin 组，为上传做好准备
5. 用 python 写一个脚本读取高通基线./repo/meanifest.xml 文件里的项目，生成创建 gerrit 对应项目，设置 gerrit 对应的组，推送代码的脚本
6. 执行脚本，中间会遇到还有 repo 过大的问题，我又写了一个脚本，把大的脚本按分支上传，一次上传一小部分，把 gerrit 超时时间加到 6 个小时。
7. 用脚本分析高通私有代码的目录，生成相关脚本来初始化 git，创建 gerrit 项目，提交代码
8. 修改 manifest 文件，增加私有代码的仓库，并把服务器地址指向自建服务器
9. 创建一个 manifest 仓库，开始测试拉取

其中 gerrit 服务器搭建参考 gerrit 搭建记录
完整流程参考 https://c55jeremy-tech.blogspot.com/2019/04/gerrit-serveraosp-codebase-part-13.html
manifest 地址 http://192.168.0.237/arparaAndroid/manifest
manifest 文件格式
Repo 目录介绍 https://gerrit.googlesource.com/git-repo/+/refs/heads/master/docs/internal-fs-layout.md
我写的脚本在这里 http://192.168.0.237/arparaAndroid/repo_creator

使用 docker 镜像 https://hub.docker.com/r/gerritcodereview/gerrit/

把这个文件命名为 docker-compose.yml

执行

```bash
Sudo useradd gerrit
Sudo mkdir -p /var/gerrit
Sudo chmod 771 /var/gerrit
Sudo chown :gerrit -R /var/gerrit
```

然后执行 docker-compose up -d 就行了

```yaml
version: '3'

services:
  gerrit:
    image: gerritcodereview/gerrit
    ports:
      - "29418:29418"
      - "80:8080"
    expose:
      - "8080"
    depends_on:
      - ldap
    volumes:
      - /var/gerrit/etc:/var/gerrit/etc
      - /var/gerrit/git:/var/gerrit/git
      - /var/gerrit/db:/var/gerrit/db
      - /var/gerrit/index:/var/gerrit/index
      - /var/gerrit/cache:/var/gerrit/cache
    environment:
      - CANONICAL_WEB_URL=http://192.168.1.134
      - WEBURL=http://192.168.1.134
       # command: init
    user: gerrit

  ldap:
    image: osixia/openldap
    ports:
      - "389:389"
      - "636:636"
    environment:
      - LDAP_ADMIN_PASSWORD=yourpassword
    volumes:
      - /var/gerrit/ldap/var:/var/lib/ldap
      - /var/gerrit/ldap/etc:/etc/ldap/slapd.d

  ldap-admin:
    image: osixia/phpldapadmin
    ports:
      - "6443:443"
    expose:
      - "6443"
    environment:
      - PHPLDAPADMIN_LDAP_HOSTS=ldap

```