---
title: 使用 debootstrap 安装 Debian 系统
date: 2023-02-14 16:21:35
tags: Debian
---

对于想要进入自由开源之世界的人来说，使用一个自由开源的操作系统应该算是第一步。在 Linux 世界中，deb 系广为人知——许多人通过 Ubuntu 了解到了其上游 Debian。

我喜欢独立的发行版本。在搜集相关信息时，为 [Debian 的理念](https://www.debian.org/intro/philosophy) 所折服。Debian 支持 [多种架构](https://wiki.debian.org/SupportedArchitectures)，尽最大努力做到其宣言的 “通用操作系统” 这一点，让我感到由衷敬佩。由此，我放弃了一开始由老师推荐安装在虚拟机中的 [Ubuntu](https://www.ubuntu.com) 操作系统，转而直接使用其上游 Debian，并逐步替换了原先物理机上使用的 Windows。

本文介绍了使用 [debootstrap](https://wiki.debian.org/Debootstrap) 安装 Debian unstable 系统的方法。采用这种方法可以得到一个尽可能简洁的系统且允许用户做自定义设置。

部分过程来源参考：[ArchLinuxStudio Arch Linux 安装使用教程 基础安装](https://archlinuxstudio.github.io/ArchLinuxTutorial/#/rookie/basic_install)。在文中对应部分采用 “来源 ArchLinuxStudio” 标注。

## 准备

1. Debian 启动盘。启动后通过 `sudo su` 进入超级用户模式。

    国内用户可以去镜像站下载 **带非自由固件的** 镜像，保证系统启动后能够正常连接网络。
    
    在此推荐使用 KDE Live 镜像和 GNOME Live 镜像。

    [USTC open source software mirror](http://mirrors.ustc.edu.cn/)

    本教程安装的系统采用 grub UEFI 启动，在引导启动盘前需先检查主机是否开启 UEFI/GPT 启动模式，进入 Live 系统后亦可执行必要的检查，具体可以参考：[GrubEFIReinstall](https://wiki.debian.org/GrubEFIReinstall)。

2. Debian 内安装：`debootstrap`，`arch-install-scripts`（为保证下载速度，国内用户需要改换软件源：[Debian 源使用帮助](http://mirrors.ustc.edu.cn/help/debian.html)）。

    ```text
    apt install debootstrap arch-install-scripts
    ```

## 磁盘处理、分区、挂载

首先确认磁盘，使用 `lsblk` 列出磁盘设备，然后 `parted` 进行分盘。
```text
# 来源 ArchLinuxStudio
lsblk                       #显示分区情况 找到你想安装的磁盘名称
parted /dev/sdx             #执行parted，进入交互式命令行，进行磁盘类型变更
(parted)mktable             #输入mktable
New disk label type? gpt    #输入gpt 将磁盘类型转换为gpt 如磁盘有数据会警告，输入yes即可
quit                        #最后quit退出parted命令行交互
```

接着使用 `cfdisk` 对磁盘进行分区，一般设置 efi 为第一个分区，类型为 `EFI System`：

```text
# 来源 ArchLinuxStudio
cfdisk /dev/sdx #来执行分区操作,分配各个分区大小，类型
fdisk -l #分区结束后， 复查磁盘情况
```

接着格式化磁盘：

```text
# 来源 ArchLinuxStudio
mkfs.ext4  /dev/sdax            #格式化根目录和home目录的两个分区
mkfs.vfat  /dev/sdax            #格式化efi分区
```

然后，挂载磁盘。先挂载根目录，然后再挂载其他目录：

```text
# 来源 ArchLinuxStudio，有更改
mount /dev/sdax  /mnt
mkdir /mnt/boot/efi     #创建efi目录, Debian 目前支持的路径如此
mount /dev/sdax /mnt/boot/efi
mkdir /mnt/home    #创建home目录
mount /dev/sdax /mnt/home
```
## 安装 base-system

使用如下命令安装 Debian unstable（sid）基础系统：

```text
debootstrap --include linux-image-amd64,locales,grub-efi,zstd --arch amd64 unstable /mnt https://mirrors.ustc.edu.cn/debian
```

`--include` 手动指定了用户想要在基础系统之上安装的包，`--arch` 指定了安装的架构，`unstable` 指定了安装的版本（不稳定版），`/mnt` 为安装位置，即之前挂载的根目录；最后的网址为镜像源地址，这里选择了中科大的镜像源，也可以换其他源。

之后等待一段时间，基础系统即安装完毕。

使用 `genfstab` （来自 `arch-install-scripts`）生成 `fstab`：

```text
genfstab -U /mnt >> /mnt/etc/fstab
```

然后，使用 `arch-chroot` 进入系统：

```text
arch-chroot /mnt
```

## 安装基础软件、配置

先打开 `/etc/apt/sources.list`，增加 `contrib`、`non-free` 和 `non-free-firmware` 仓库：

```text
nano /etc/apt/sources.list
```

编辑好的地址如下所示：

```text
deb https://mirrors.ustc.edu.cn/debian unstable main contrib non-free non-free-firmware
```

<font color=red>\# 根据近期（2022）的一项 <a href="https://www.debian.org/vote/2022/vote_003#outcome">投票</a>，Debian 将在最新的镜像中包含非自由固件，用户可以选择是否启用对应的 `non-free-firmware` 新仓库。据此，已在 unstable 的用户如有需要，需在 `sources.list` 中新增 `non-free-firmware` 字段，启用新仓库以收到后续的非自由固件更新。</font>

然后，更新缓存：

```text
apt update
```

安装如下软件：

```text
apt install zsh  # Z shell

apt install sudo # 超级用户权限获取工具

apt install neovim # 一个文本编辑器
```

安装固件（<font color=red>注意：`firmware-iwlwifi` 和 `firmware-realtek` 均属于非自由固件</font>）：

```text
# iwlwifi 提供了 Intel 网卡和蓝牙的支持，如果有 realtek 网卡则需要安装 firmware-realtek，需要事先检查
apt install firmware-linux firmware-iwlwifi
apt install firmware-realtek
```

设置主机名相关（来源 ArchLinuxStudio）：

首先在 `/etc/hostname` 中加入主机名，如叫 debian。（用 Ubuntu 镜像安装时，会复制主机的主机名配置等，因此会看到 `ubuntu` 字样，可以改掉）。

```text
nvim /etc/hostname
```

接着在 `/etc/hosts` 中配置对应的条目：

```text
127.0.0.1   localhost
::1         localhost
127.0.1.1   debian  # 这里要填入刚刚设置的主机名
```

设置 `locales` 和时区：

```text
dpkg-reconfigure locales
dpkg-reconfigure tzdata
```

locales 建议选择英文（`en_US.UTF-8`）。时区选择 `Asia/Shanghai`。

随后新增普通用户，加入 sudoers，并设置密码：

首先为 root 用户设置密码：

```text
passwd
```
然后新增普通用户，假设用户名叫 joe：

```text
useradd -m -s /bin/zsh joe
```

加入 `sudoers`：

```text
usermod -aG sudo joe
```

为 joe 设置密码：

```text
passwd joe
```

修改默认的终端为 zsh：

```text
chsh
# 随后会询问新 sehll 的位置，输入 /bin/zsh
```

最后，安装引导：

```text
grub-install --target=x86_64-efi --efi-directory=/boot/efi
```

更新 grub 启动条目：

```text
update-grub
```

至此，一个基础的、有命令行界面的 Debian unstable 已经安装好了。接下来继续操作，准备安装桌面环境。


## 安装桌面环境（Desktop Environment, DE）与其他软件

关于桌面环境，网络上有许许多多的介绍和建议。Debian 提供了多种桌面环境的打包，可以自由选择。不过主流的应当是三个：KDE Plasma，GNOME，Xfce。

当然，也可以选择只安装 [窗口管理器（Window Manager, WM）](https://wiki.debian.org/WindowManager)。网上有许多平铺桌面管理器的美化教程，可以先通过视频了解一下 WM 的效果。这部分需要自己去琢磨如何配置，安装必要的软件了。本文介绍安装桌面环境。

依然从之前的 Ubuntu 系统通过 `arch-chroot` 进入 Debian chroot 环境开始：

以安装 [GNOME](https://www.gnome.org) 桌面环境为例：

```text
# gnome-shell GNOME 桌面环境图形部分，提供了窗口切换等基础功能
# nautilus    GNOME 文件管理器
# gnome-terminal  GNOME 终端模拟器
# gdm3            GNOME Display Manager，提供桌面登陆功能并会做初始环境配置等

apt install gnome-shell nautilus gnome-terminal gdm3
```

接下来还有其他一些软件可以一并装上：

```text
apt install firefox    # 火狐浏览器
apt install rhythmbox  # 为 GNOME 设计的音乐播放器
apt install eog        # GNOME 之眼图像查看器
apt install evince     # 文档查看器
apt install mpv        # mpv 视频播放器
```
至此，GNOME 桌面环境和基础软件全部安装完毕。可以重启后进入新系统：

```text
exit # 退出 chroot
reboot # 重启
```
给出自己的桌面展示：

![GNOME Desktop](https://pics.galiren.me/gnome-desktop.png)

最后，虽然这话可能不该由我来说，但我依然要向看这篇小文且做到最后的读者表示衷心祝贺：欢迎进入 Debian 的世界！
