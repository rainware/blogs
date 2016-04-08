---
layout: post
title:  "为linux系统挂载硬盘"
date:   2016-04-07 15:01:00 +0800
category: 安装文档
tagline: "Supporting tagline"
tags: [linux, mount, fdisk, ubuntu]
---

# 1. 分区格式化

硬盘有 disk size 和 partition size 两个概念。

如果硬盘是第一次加载的硬盘，就需要进行分区、格式化，和 mount 操作。 如果是老硬盘，且没有扩容，就不用再分区、格式化了，直接 mount 就行。

```
 如果硬盘容量大于1TB，不要用 fdisk，可使用 parted 工具进行分区。
```

以 Ubuntu Linux 为例，以下操作需要 root 权限。

### 1.1 分区

有两种分区方式, `fdisk` 和 `parted`

##### 1.1.1 fdisk 分区

通过 `fdisk -l` 命令查看挂载的硬盘, 假设为 `/dev/vdc`

```bash
$ fdisk -l
...
Disk /dev/vdc: 10.7 GB, 10737418240 bytes
64 heads, 32 sectors/track, 10240 cylinders, total 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/sdc doesn't contain a valid partition table
```
