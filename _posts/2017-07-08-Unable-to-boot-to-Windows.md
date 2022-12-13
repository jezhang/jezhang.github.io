---
layout:     post
title:      "Windows10启动进入grub命令行解决方法"
date:       2017-06-06 21:44:00
author:     "jezhang"
header-img: "https://asdfex.oss-cn-qingdao.aliyuncs.com/jezhang.github.io/img/post-bg-2015.jpg"
catalog: true
tags:
    - Linux
    - Windows
---


### 情况说明
系统安装了windows 10，linux mint双系统。
在windows10中删除了linux lint所在的分区，以及对应的EFI分区。
重新开机显示gnu grub命令行界面。
输入exit，重启，还是进入gnu grub命令行界面。
适用于删除 grub 之前删除了 Ubuntu 分区的 EFI。

### 解决方法
在grub命令行输入以下命令即可进入windows 10：

```sh
insmod part_gpt
insmod chain
set root=(hd0,gpt1)
chainloader /EFI/Microsoft/Boot/bootmgfw.efi
boot
```
在命令行grub模式下，ls会列出硬盘分区，help列出可用命令。
需要为 set root= 命令输入EFI引导分区（而不是windows分区）的ID
只要正确设置了根目录，就可以使用ls / 命令查看文件和目录以找到Windows启动管理器的正确路径。（一般均为默认路径）

### 进入win10后删除gnu grub



