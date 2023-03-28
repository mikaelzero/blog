---
title: Gerrit搭建和使用
urlname: gerrit-build-and-use
date: 2023/03/28
password: helloasdkjhasfisfw
tags:
  - Gerrit
  - Tools
---

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