---
layout: post
title:  "在Ubuntu上搭建NFS"
date:   2016-04-15 17:42:00 +0800
category: 安装文档
tagline: "Supporting tagline"
tags: [linux, mount, 格式化, 分区, nfs, 网盘, fdisk, ubuntu]
---

在服务器众多的情况下, 很多时候我们希望我们只在一台服务器上做操作(拉取代码, 上传文件), 而其他服务器上能自动共享这些文件.

这里介绍使用NFS来为服务器挂载网盘.

# 1. 安装

NFS server和 NFS client上都需要安装nfs, ubuntu下使用apt-get进行安装

```
$ apt-get install nfs-kernel-server
```

# 2. NFS server

选取一台主机作为NFS server, 选取一个目录作为共享目录, 这样以后我们在该目录下的做的文件操作, 就能共享到其他的主机上了.

* 创建共享目录

```
$ mkdir /tatafile
```

* 编辑配置文件 `vim /etc/exports`, 加入如下配置

```
/tatafile 192.*.*.*(rw,sync,no_root_squash,no_subtree_check)
```

> * 192.\*.\*.\*: 只允许192开头的ip来访问
> * ro: 共享目录只读
> * rw: 共享目录可读可写
> * sync: 将数据同步写入内存缓冲区与磁盘中，效率低，但可以保证数据的一致性
> * async: 将数据先保存在内存缓冲区中，必要时才写入磁盘
> * no_root_squash: 登入 NFS client 主机使用分享目录的使用者，如果是 root 的话，那么对于这个分享的目录来说，就具有 root 的权限(不是很安全的配置)
> * root_squash(默认): 将来访的root用户映射为匿名用户或用户组
> * no_subtree_check: 即使输出目录是一个子目录，nfs服务器也不检查其父目录的权限，这样可以提高效率

* 重启rpcbind

```
$ service rpcbind restart
```

* 重启nfs服务

每次修改/etc/exports后都需要重启nfs

```
$ service nfs-kernel-server restart
```

# 3. NFS client

* 创建共享目录

```
$ mkdir /tatafile
```

* 挂载网盘

```
$ mount -t nfs <NFS server ip-address>:/tatafile /tatafile
```

* 设置开机自动挂载

把指令 `mount -t nfs <NFS server ip-address>:/tatafile /tatafile` 写到 `/etc/rc.local` 中即可.
