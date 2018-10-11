---
layout: article
title: Linux内存占满释放
tags: Linux
---

> Linux为了提高系统效率，会把部分使用过的文件缓存到内存中，长此以往会造成内存消耗过多。

<!--more-->

首先执行`sync`命令，刷新文件系统的缓存，将所有未写的系统缓冲区写到磁盘中。
清理内存的命令可分为三个层次如下：
1. 只清理页面缓存
```shell
echo 1 > /proc/sys/vm/drop_caches
```
2. 清理目录项(dentry)和索引节点(inode)
```shell
echo 2 > /proc/sys/vm/drop_caches
```
3. 清理所有缓存
```shell
echo 3 > /proc/sys/vm/drop_caches
```

上述操作不会影响正在运行的应用或服务。

在实际生产环境中，建议使用第一条命令，第三条命令会清除包括页面缓存，目录项和索引节点在内的所有内存缓存，影响系统效率。



参考文档：https://www.tecmint.com/clear-ram-memory-cache-buffer-and-swap-space-on-linux