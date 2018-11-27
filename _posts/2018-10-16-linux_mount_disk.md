---
layout: article
title: Linux挂载新添加的硬盘
tags: Linux
---

> 对Linux已有目录扩容，挂载新的硬盘

<!--more-->
**注意：挂载新的硬盘会格式化目标目录，需对目录下的内容做备份。**

```shell
# 查看硬盘分区
fdisk -l 
# 分区 
fdisk /dev/vdc 
# 输入n新建一个分区
Command (m for help):n
# 输入p建立主分区
Command action
   e   extended
   p   primary partition (1-4)
# 选择分区个数，输入 1
Partition number(1-4)：1
# 指定分区起始位置
First cylinder(1-175664， default 1)：1
# 指定分区结束位置
Last cylinder or + size or sizeM or + sizeK(1-2080507,default 2080507): 
Using default value 2080507 
# 查看分区
Command (m for help): p
# 写保存并退出
Command (m for help): w
# 查看是否已经建好逻辑磁盘
fdisk -l
# 建立文件系统
mkfs.ext4 /dev/vdc1
# 将做好的文件系统挂载到 target 文件夹上
mount /dev/vdc1 /target/
```

挂载完成后，使用`df -h`命令可以查看硬盘挂载情况

最后，修改`/etc/fstab`文件，添加如下内容，设置开机自动挂载
```shell
/dev/vdc1               /target            ext4    defaults        0 0
```

**参考资料：**

<https://blog.csdn.net/jiandanjinxin/article/details/69969217>

<https://www.jianshu.com/p/37a550cc0371?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation>

<https://jaywcjlove.gitee.io/linux-command/c/fdisk.html>