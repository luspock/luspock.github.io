---
title: 树莓派3挂载U盘安装Archlinux
date: 2017-10-28 15:58:41
categories:
  - 硬件
  - 树莓派
tags:
  - 折腾
  - 树莓派
---

## 前言

暑假的时候入手了一个树莓派3，配了一个32G的SD 卡。由于正事比较多，在安装系统上拖了很长时间。目前树莓派上面的系统已经能够正常运行，于是在这里整理记录一下安装过程，以对自己和其他人有所帮助。

## 选择Archlinux Arm

自己之前就折腾过一段时间Archlinux，当时在笔记本上安装了Win10和Archlinux双系统。因为平时主要使用Windows，对Arch刚需不大，安装完成之后就很少进入Arch。后面由于SSD空间不够就把Archlinux删去了。入手的树莓派刚好可以让我重新接触这个轻量级，简洁的系统。
<!-- more -->
## 安装准备

* 树莓派 
* 32G大小SD卡（由于只在上面安装Boot,其实30M+就可以了）
* USB读卡器
* Archlinux ARM安装完成之后会占用600-800M空间，再安装一点软件就要1.5G左右，因此最好准备4G以上的U盘

{% asset_img space_df_h.JPG %}

* 路由器

## 安装过程

之前Raspberry Pi官网上是有Archlinux Arm的镜像文件的，但现在树莓派官方取消支持了。Archlinux Arm还是给出安装的具体过程。

但是步骤操作起来比较麻烦，我采用的是镜像文件安装的方法，ta人维护的镜像文件可以见这里[image](https://sourceforge.net/projects/archlinux-rpi2/)，请谨慎使用。

1. 用Win32DiskImager 将镜像写入SD 和U盘（先格式化成FAT格式）。

2. 将SD卡和U盘同时插到树莓派上，开机。将树莓派和电脑接到同一个路由器下，方便获得IP地址。用IP扫描器或者登陆路由器管理界面查看树莓派的IP地址（alarmpi）。ssh登陆树莓派，用户名alarm，密码alarm。su切换到root，密码root。

3. lsblk -l 查看U盘编号，我这里的情况是sda

4. 由于写入镜像的时候，系统只占用一部分空间，因此需要扩展分区。

   `fdisk /dev/sda`

   按`p`显示分区信息，记住/sda2开始的位置

   按`d`删掉第二个分区

   按`n`然后`p`新建主分区，正常情况下新分区开始的位置是刚才记住的/sda2开始位置，按两次`enter`完成分区创建

   按`p`确认第二个分区使用了剩下的所有空间

   按`w`将新的分区信息保存，如果提示是否覆盖xxx, 选择no!否则会丢失sda2上的image数据。

5. 修改SD卡的/boot中cmdline.txt

   `root=/dev/mmcblk0p2` 改为 `root=/dev/sda2`

6. `reboot`重启过后正常情况下进入的是U盘里面的系统，用户名alarm,密码alarm。su切换到root，密码root。

7. `df -h`查看挂载信息，重新调整大小后，发现sda2大小依然没有变化

   `sudo resize2fs /dev/sda2` 更新系统，如果可能还需reboot。


## 安装后配置

### 添加用户

`su`切换到root，`passwd`更新root密码

`useradd -m -g wheel -s /bin/bash username`

`passwd`设置新用户密码

重新用新的用户登陆后再切换到root

`userdel -r alarm`删除默认的alarm用户

### 更改源

`nano /etc/pacman.d/mirrorlist`

添加清华tuna源`Server = http://mirrors.tuna.tsinghua.edu.cn/archlinuxarm/$arch/$repo`

添加USTC源`Server = http://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch`

###更新系统

`pacman-key init`

`pacman -Syu`更新系统（时间较长）

如果提示`/boot/overlays/audioinjector-addons.dtbo exists in filesystem`，`rm`相应文件，然后再次`pacman -Syu`即可

### 安装sudo

`pacman -S sudo`

root身份运行`visudo`，将`%wheel ALL=(ALL) ALL`前面的#注释去掉

### 修改ssh

`nano /etc/ssh/sshd_config`

修改port端口`Port xxxx`

禁止root登陆`PermitRootLogin no`

`systemctl reload sshd`

先不要断开目前的连接，再开一个新的连接，用更新过后的配置测试通过再断开当前连接为好。

### 安装软件

`sudo pacman -S vim git tmux base-devel screenfetch`

`sudo pacman -S python3 python-pip`

## 总结

{% asset_img screenfetch.JPG %}

介绍了在树莓派上挂载U盘安装archlinux过程以及简要配置，之后会说明如何将树莓派配置成一个wifi热点。


