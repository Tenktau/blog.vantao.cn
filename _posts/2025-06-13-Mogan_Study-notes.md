---
title: 「墨干理工套件」初次尝试编译过程笔记
description: 本篇文章旨在分享本人初次尝试编译墨干理工套件过程中的心得。
date: 2025-06-13 +0800
categories: [Development,Software]
tags: [Mogan,Linux]
---

> **虚拟机软件**：VirtualBox 7.0.10 r158379 (Qt5.15.2)  
> **客户机**：Debian 12.11.0 (Linux 6.1.0)  
> **宿主机**：Window 10（版本号：2009）

## 1 编译环境配置

在 VirtualBox 中安装完 Debian 之后，根据 Mogan 官方教程进行了编译环境的配置，在这个过程中遇到的问题记录、解决方案与参考资料如下：

### 1.1 无法添加第三方源

首先，在执行 `sudo add-apt-repository ppa:xmake-io/xmake` 命令后产生如下报错：

```bash
tenktau@10:~/Desktop/mogan$ sudo add-apt-repository ppa:xmake-io/xmake
Traceback (most recent call last):
  File "/usr/bin/add-apt-repository", line 362, in <module>
    sys.exit(0 if addaptrepo.main() else 1)
                  ^^^^^^^^^^^^^^^^^
  File "/usr/bin/add-apt-repository", line 345, in main
    shortcut = handler(source, **shortcut_params)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3/dist-packages/softwareproperties/shortcuts.py", line 40, in shortcut_handler
    return handler(shortcut, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3/dist-packages/softwareproperties/ppa.py", line 86, in __init__
    if self.lpppa.publish_debug_symbols:
       ^^^^^^^^^^
  File "/usr/lib/python3/dist-packages/softwareproperties/ppa.py", line 126, in lpppa
    self._lpppa = self.lpteam.getPPAByName(name=self.ppaname)
                  ^^^^^^^^^^^
  File "/usr/lib/python3/dist-packages/softwareproperties/ppa.py", line 113, in lpteam
    self._lpteam = self.lp.people(self.teamname)
                   ^^^^^^^^^^^^^^
AttributeError: 'NoneType' object has no attribute 'people'
```

通过检索得知其本质是 PPA 源管理工具部分缺失，于是通过 `sudo apt-get install software-properties-common` 成功解决以上报错。

然后，即便消除了报错，并且成功添加 PPA 源之后，在执行 `sudo apt install --yes build-essential libfontconfig1-dev  qt6-base-dev libqt6svg6-dev qt6-image-formats-plugins libcurl4-openssl-dev libfreetype-dev libgit2-dev zlib1g-dev libssl-dev libjpeg-turbo8-dev cmake` 时仍然会有找不到包的情况，于是我选择手动安装对应的 `lib*` 或 `*-dev` 包（[参考资料](https://unix.stackexchange.com/questions/330068/package-fontconfig-not-found-despite-having-installed-libfontconfig1-dev)），最终成功消除报错，并且进入编译阶段。

需要注意的是，如果此处选择自己手动安装缺失的软件包，后续可能找不到 Qt（有可能压根没安装），我是采用了力大砖飞的解决办法，即通过 `sudo apt install qt6*` 安装所有 Qt6 相关的包。

最终按照官方指导步骤编译成功，如下所示：

![](/assets/img/250613_mogan-build-success.png)