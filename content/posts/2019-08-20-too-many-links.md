---
title: Too many links problem
author: fangfeng
date: 2019-08-19
categories:
  - 技术
tags:
  - docker 
  - fs
  - ext4
---

今天使用 Kubernetes 发布 Deployment 时 Pod 拉取镜像频繁失败。查看日志发现报错信息如下：

```sh
Warning  Failed 4m  kubelet, 172.16.1.1 Failed to pull image "xxx.xxx.com/<image-name>:<version>": rpc error: code = Unknown desc = failed to register layer: link /data/docker/storage/overlay/48420e2e1cacbf1e5a0766f892556b6b5aed9b86bbc118fb4ab5b1f236bcb55a/root/var/lib/yum/yumdb/o/e23ec6b5d5de041869ec5f3e436685acce4a7354-openssh-server-7.4p1-13.el7_4-x86_64/checksum_type /data/docker/storage/overlay/7545256b7c82ea95260e05f3eb0ac6b4138e9f206febd09f82863f1ef17575da/tmproot525452492/var/lib/yum/yumdb/o/e23ec6b5d5de041869ec5f3e436685acce4a7354-openssh-server-7.4p1-13.el7_4-x86_64/checksum_type: too many links
```

最初还以为是网络的问题，但出现得实在是太过于频繁。仔细观察发现每次 Pod 拉取镜像失败，只会发生在特定的一台 K8s Node 上 `172.16.1.1`。而 Kubernetes 事件中提示的是 `too many links`，对问题原因的怀疑开始转向文件系统。

<!--more-->

## Inode 与硬链接

先验知识请参看[文件系统](https://www.ffutop.com/posts/2018-10-14-understand-kernel-5/)。

Inode 是文件系统描述一段数据内容的核心结构。目录下的文件名/目录名通过指向一个特定的 Inode 来绑定到数据内容上。同一个 Inode 可以有很多很多个与众不同的文件名/目录名，只需要保证这些文件/目录最终指向同一个 Inode，这对应的概念也就是“**硬链接**”。

文件系统的硬链接数有上限吗？显然是有的，这个链接数总是需要一个字段来存储的。首先 CHECK 下使用的文件系统。

```sh
[root@172-16-1-1]$ mount
...
/dev/sdc on /data/docker/storage/overlay type ext4 (rw,relatime,seclabel)
```

再 CHECK 下出问题的文件，**64854**，差不多等同于 $2^{16}$。

```sh
[root@172-16-1-1]$ ls -alh checksum_type
-rw-r--r--. 64854 root root   64 Fed  2  2018  checksum_type
```

查查 EXT4 的硬链接上限

```c
/* fs/ext4/ext4.h */

/*
 * Maximal count of links to a file
 */
#define EXT4_LINK_MAX       65000

struct ext4_inode {
    __le16  i_mode;     /* File mode */
    __le16  i_uid;      /* Low 16 bits of Owner Uid */
    __le32  i_size_lo;  /* Size in bytes */
    __le32  i_atime;    /* Access time */
    __le32  i_ctime;    /* Inode Change time */
    __le32  i_mtime;    /* Modification time */
    __le32  i_dtime;    /* Deletion Time */
    __le16  i_gid;      /* Low 16 bits of Group Id */
    __le16  i_links_count;  /* Links count */
    __le32  i_blocks_lo;    /* Blocks count */
    __le32  i_flags;    /* File flags */
    /* ... some things omitted ... */
}
```

`i_links_count` 字段用来描述 Inode 被引用的次数，类型是 `__le16`，小端存储的 16 位数。又有一个宏“最大 EXT4 链接数”，硬编码的上限值被定在了 65000

OK，为什么还差了 146 个硬链呢？按照逻辑应该能够再添加几个。且耐心往下看。

## Overlay File System

首先确认下 `docker` 使用的存储驱动，

```sh
[root@172-16-1-1]$ docker info 
# ... some thing omitted ...
Storage Driver: overlay
 Backing Filesystem: extfs
 Supports d_type: true
```

使用了 Overlay 驱动；overlay 使用硬链接的方式完成镜像的叠加，这部分也算是这个 UnionFS 和 Docker 组合使用下的硬伤了。

![](https://docs.docker.com/storage/storagedriver/images/overlay_constructs.jpg)

最后检查一下这个文件被哪些文件链接吧。

```sh
[root@172-16-1-1]$ ls -i checksum_type 
34997956 /data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/6e92fab82fc8215308e066f5f984fb4fcf26a404-e2fsprogs-1.42.9-10.el7-x86_64/checksum_type
[root@172-16-1-1]$ cd /data/docker/storage/overlay/
[root@172-16-1-1]$ find . -inum 34997956 | wc 
64854
```

恰好符合之前得到的硬链接数量。最后的最后，就是为什么离上限还差了百余链接数。实际上看一下 `find . -inum 34997956` 打印出的前几项就能得出结论了。

```sh
[root@172-16-1-1]$ find . -inum 34997956 
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/6e92fab82fc8215308e066f5f984fb4fcf26a404-e2fsprogs-1.42.9-10.el7-x86_64/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/50be40805794126a56fecc7f720a00eddbeb0ac4-ebtables-2.0.10-15.el7-x86_64/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/6afbf327df2e805ec7f41172744151395afefb4b-elfutils-default-yama-scope-0.168-8.el7-noarch/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/cdde997a3984c69574975c4516593744d0578fe5-e2fsprogs-libs-1.42.9-10.el7-x86_64/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/3e23a7d10536c7794ef133302f2213f3eca11028-elfutils-libs-0.168-8.el7-x86_64/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/7e621968f73154565600b86be2fbc162cc6e0f47-ethtool-4.8-1.el7-x86_64/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/838dfbc19277a0435fd7bf182149d624dc4d2ecf-elfutils-libelf-0.168-8.el7-x86_64/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/e/60ff81da46672c6d0409df1180d5076641c50934-expat-2.1.0-10.el7_3-x86_64/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/G/5d1d730c5f843277da5723b9d79e05fa10cd8019-GeoIP-1.5.0-11.el7-x86_64/checksum_type
/data/docker/storage/overlay/071d913f222a78d3fd42e371ae506cd44ce134514b80a9045a787f01cad74d30/root/var/lib/yum/yumdb/g/a20c407739d081345b8b2ab34d4d9e5f8e054f65-glib2-2.50.3-3.el7-x86_64/checksum_type
# ... more omitted ...
```

`checksum_type` 这个文件在单一镜像中就已经被链接了近两百次，这就意味了一个新的 Layer 的创建必然带来新的200+链接，在创建过程中也就必然失败了。

## 解决方案?

替掉 overlay，换上 overlay2 是官方最为推荐的方案了。毕竟这个问题是 overlay 设计上引起的问题，是一种必然。

临时性的解决方案就是删除掉不再使用的镜像: `docker system prune -a`
