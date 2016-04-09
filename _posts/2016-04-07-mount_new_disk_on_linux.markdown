---
layout: post
title:  "为linux系统挂载硬盘"
date:   2016-04-07 15:01:00 +0800
category: 安装文档
tagline: "Supporting tagline"
tags: [linux, mount, fdisk, ubuntu]
---

转载自 *[青云FAQ](https://docs.qingcloud.com/faq/index.html#id9)*

# 1. 分区格式化

硬盘有 disk size 和 partition size 两个概念。

如果硬盘是第一次加载的硬盘，就需要进行分区、格式化，和 mount 操作。 如果是老硬盘，且没有扩容，就不用再分区、格式化了，直接 mount 就行。

>  警告 如果硬盘容量大于1TB，不要用 fdisk，可使用 parted 工具进行分区。


以 Ubuntu Linux 为例，以下操作需要 root 权限。

### 1.1 分区

有两种分区方式, `fdisk` 和 `parted`

##### 使用 fdisk 分区

通过 `fdisk -l` 命令查看挂载的硬盘, 假设为 `/dev/vdc`

```
$ fdisk -l
...
Disk /dev/vdc: 10.7 GB, 10737418240 bytes
64 heads, 32 sectors/track, 10240 cylinders, total 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/vdc doesn't contain a valid partition table
```

对硬盘进行分区

```
$ fdisk /dev/vdc
```

然后根据提示，依次输入 n, p, 1, 以及 两次回车，然后是 wq，完成保存。 这样再次通过 `fdisk -l` 查看时，你可以看到新建的分区 `/dev/vdc1`

```
# fdisk -l
...
Disk /dev/sdc: 10.7 GB, 10737418240 bytes
64 heads, 32 sectors/track, 10240 cylinders, total 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x17adb4cb

Device Boot      Start         End      Blocks   Id  System
/dev/sdc1            2048    20971519    10484736   83  Linux
```

##### 使用 parted 分区

通过 `parted -l` 命令查看新挂载的硬盘，假设为 `/dev/sdc`

```
$ parted -l
...

错误: /dev/sdc: unrecognised disk label
```

对硬盘进行分区:

```
$ parted /dev/sdc
```

然后创建新分区

```
(parted) mklabel gpt
(parted) mkpart primary 1049K -1
(parted) quit
```

这时再查看硬盘信息时会看到 /dev/sdc1

```
$ parted -l
...
Model: QEMU QEMU HARDDISK (scsi)
Disk /dev/sdc: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt

Number  Start   End     Size    File system  Name     标志
 1      1049kB  10.7GB  10.7GB               primary
```

### 1.2 格式化

例如格式化为 ext4 格式

```
$ mkfs -t ext4 /dev/sdc1
```

### 1.3 挂载硬盘

```
$ mkdir -p /mnt/sdc && mount -t ext4 /dev/sdc1 /mnt/sdc
```

> 警告 为了防止宿主机在突然断电时可能对数据带来的风险，如果文件系统是ext3， 则需要在mount的时候显式的指定”barrier=1”选项，例如”mount -t ext3 -o barrier=1 /dev/sdc1 /mnt/point”

# 2 设置自动挂载

一般在 `/etc/fstab` 中指定, 推荐使用UUID或者LABEL的方式

##### UUID方式

先通过 “blkid /dev/sdc1” 命令，得到磁盘的 UUID，例如：

```
/dev/sdc1: UUID="185dc58b-3f12-4e90-952e-7acfa3e0b6fb" TYPE="ext4"
```

然后在 `/etc/fstab` 里加入:

```
UUID=185dc58b-3f12-4e90-952e-7acfa3e0b6fb /mnt/mydisk ext4 defaults 0 2
```

##### LABEL方式

在格式化硬盘的时候需要指定LABEL:

```
mkfs -t ext4 -L MY_DISK_LABEL /dev/sdc1
```

然后在 `/etc/fstab` 里面加入:

```
LABEL=MY_DISK_LABEL /mnt/mydisk ext4 defaults 0 2
```

> 警告 修改完 fstab 请使用 “mount -a” 先检查下是否有问题。
