+++
title = "LVM 磁盘扩容"
slug = "post-21"
date = "2025-06-11T22:16:00+08:00"
description = "环境为VMware环境下CentOS 6.8"
categories = ["shell"]
+++

环境为VMware环境下CentOS 6.8

## 名词解释

**物理存储介质（The physical media）**
> 这里指系统的存储设备：硬盘，如：/dev/hda、/dev/sda等等，是存储系统最低层的存储单元。

**物理卷PV（physical volume）**
> 物理卷就是指硬盘分区或从逻辑上与磁盘分区具有同样功能的设备(如RAID)，是LVM的基本存储逻辑块，但和基本的物理存储介质（如分区、磁盘等）比较，却包含有与LVM相关的管理参数

**卷组VG（Volume Group）**
> LVM卷组类似于非LVM系统中的物理硬盘，其由物理卷组成。可以在卷组上创建一个或多个"LVM分区"（逻辑卷），LVM卷组由一个或多个物理卷（PV）组成。也可以把VG理解成一个大的仓库或者几块大的硬盘

**逻辑卷LV（logical volume）**
> 是从VG中划分的逻辑分区；LVM的逻辑卷类似于非LVM系统中的硬盘分区，在逻辑卷之上可以建立文件系统(比如/home或者/usr等)。

**PE（physical extent）**
> 每一个物理卷被划分为称为PE(Physical Extents)的基本单元，具有唯一编号的PE是可以被LVM寻址的最小单元。PE的大小是可配置的，默认为4MB。

**LE（logical extent）**
> 逻辑卷也被划分为被称为LE(Logical Extents)的可被寻址的基本单位。在同一个卷组中，LE的大小和PE是相同的，并且一一对应。

## 操作步骤

### 1. 添加新硬盘

```shell
# 添加硬盘后可以使用"fdisk -l"查看硬盘，如果找不到新硬盘可以执行以下命令：
echo "- - -" > /sys/class/scsi_host/host0/scan

# 如果仍然无法找到新硬盘，可以尝试更改host0为host1或host2
for (( i=0; i<=32; i++ )); do echo "- - -" > /sys/class/scsi_host/host$i/scan;done
```

### 2. 查看PV

```shell
[root@CentOS6 ~]# pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda2
  VG Name               vg_centos6
  PV Size               99.51 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              25474
  Free PE               0
  Allocated PE          25474
  PV UUID               tGJcRv-Fesq-qbCg-EdMv-tGUs-fHYn-ZTXM6r
```

### 3. 创建PV

新添加的硬盘为 `/dev/sdb`

```shell
[root@CentOS6 ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created
```

### 4. 扩容VG

```shell
[root@CentOS6 ~]# vgextend vg_centos6 /dev/sdb
  Volume group "vg_centos6" successfully extended
```

### 5. 查看LV

```shell
[root@CentOS6 ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/vg_centos6/lv_root
  LV Name                lv_root
  VG Name                vg_centos6
  LV Size                50.00 GiB
```

### 6. 扩容LV

```shell
[root@CentOS6 ~]# lvextend -l +100%FREE /dev/vg_centos6/lv_root
  Size of logical volume vg_centos6/lv_root changed from 50.00 GiB (12800 extents) to 70.00 GiB (17919 extents).
  Logical volume lv_root successfully resized.
```

### 7. 重新设置文件系统大小

ext4 格式：

```shell
[root@CentOS6 ~]# resize2fs /dev/vg_centos6/lv_root
resize2fs 1.41.12 (17-May-2010)
Filesystem at /dev/vg_centos6/lv_root is mounted on /; on-line resizing required
Performing an on-line resize of /dev/vg_centos6/lv_root to 18349056 (4k) blocks.
The filesystem on /dev/vg_centos6/lv_root is now 18349056 blocks long.
```

xfs 格式：

```shell
[root@CentOS6 ~]# xfs_growfs /dev/centos/root
```

## 扩容完成

扩容操作全部完成，逻辑卷已从 50GiB 扩展到 70GiB。