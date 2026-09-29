# Ceoh 集群 OSD Down 问题解决


# 查看问题

记录一次 Ceph OSD down 的问题，将故障 OSD 对应磁盘还原成裸盘重新接入集群

## 查看 OSD 状态

```bash
ceph osd status
ceph osd tree

#查看osd对应的硬盘
ceph device ls-by-daemon osd.0
```

## 重启有问题的 OSD

```bash
systemctl status ceph-6b6d577c-b035-11ee-8b7f-e51965640d97@osd.0.service
systemctl restart ceph-6b6d577c-b035-11ee-8b7f-e51965640d97@osd.0.service
```

## 查看启动日志

```plaintext
2024-03-13T01:57:33.588+0000 7f510481f640 0 osd.0:4.OSDShard using op scheduler mClockScheduler
2024-03-13T01:57:33.756+0000 7f510481f640 1 bdev(0x5573cf796000 /var/lib/ceph/osd/ceph-0/block) open path /var/lib/ceph/osd/ceph-0/block
2024-03-13T01:57:33.760+0000 7f510481f640 1 bdev(0x5573cf796000 /var/lib/ceph/osd/ceph-0/block) open size 107369988096 (0x18ffc00000, 100 GiB) block_size 4096 (4 KiB) rotational device, discard supported
2024-03-13T01:57:33.896+0000 7f510481f640 1 bluestore(/var/lib/ceph/osd/ceph-0) _set_cache_sizes cache_size 1073741824 meta 0.45 kv 0.45 data 0.06
2024-03-13T01:57:33.896+0000 7f510481f640 1 bdev(0x5573cf796380 /var/lib/ceph/osd/ceph-0/block) open path /var/lib/ceph/osd/ceph-0/block
2024-03-13T01:57:33.896+0000 7f510481f640 1 bdev(0x5573cf796380 /var/lib/ceph/osd/ceph-0/block) open size 107369988096 (0x18ffc00000, 100 GiB) block_size 4096 (4 KiB) rotational device, discard supported
2024-03-13T01:57:33.896+0000 7f510481f640 1 bluefs add_block_device bdev 1 path /var/lib/ceph/osd/ceph-0/block size 100 GiB
2024-03-13T01:57:33.896+0000 7f510481f640 1 bluefs mount
2024-03-13T01:57:34.020+0000 7f510481f640 1 bluefs _init_alloc shared, id 1, capacity 0x18ffc00000, block size 0x10000
2024-03-13T01:57:34.432+0000 7f510481f640 1 bluefs mount shared_bdev_used = 0
2024-03-13T01:57:34.436+0000 7f510481f640 1 bluestore(/var/lib/ceph/osd/ceph-0) _prepare_db_environment set db_paths to db,102001488691 db.slow,102001488691
2024-03-13T01:57:34.756+0000 7f510481f640 -1 bluestore(/var/lib/ceph/osd/ceph-0) _open_db erroring opening db:
2024-03-13T01:57:34.756+0000 7f510481f640 1 bluefs umount
2024-03-13T01:57:34.756+0000 7f510481f640 1 bdev(0x5573cf796380 /var/lib/ceph/osd/ceph-0/block) close
2024-03-13T01:57:34.936+0000 7f510481f640 1 bdev(0x5573cf796000 /var/lib/ceph/osd/ceph-0/block) close
2024-03-13T01:57:35.052+0000 7f510481f640 -1 osd.0 0 OSD:init: unable to mount object store
2024-03-13T01:57:35.052+0000 7f510481f640 -1 ** ERROR: osd init failed: (5) Input/output error
```

可以看到这里日志报警 Bluestore 无法打开数据库，报告了“Input/output error” (I/O 错误)
可能的原因:

- 磁盘/分区故障： osd.0 运行所在的磁盘或分区可能出现物理故障，导致读写操作失败
- 文件系统损坏： osd.0 使用的文件系统可能因各种原因出现损坏
- 配置错误： Ceph 的配置可能存在问题，导致 OSD 无法找到或正确使用存储设备
- 权限问题： Ceph 进程可能缺少对 /var/lib/ceph/osd/ceph-0 目录及其中文件的必要权限

# 排错过程

配置是没有更改的，应该不是配置的问题

## 检查磁盘状态

```bash
smartctl -a /dev/nvme1n1
```

使用 SMART 进行磁盘检测，发现磁盘本身无问题

```plaintext
smartctl 7.0 2018-12-30 r4883 [x86_64-linux-5.19.7-1.el7.elrepo.x86_64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Number:                       Amazon Elastic Block Store
Serial Number:                      vol0d9d42104245d5125
Firmware Version:                   2.0
PCI Vendor/Subsystem ID:            0x1d0f
IEEE OUI Identifier:                0xa002dc
Controller ID:                      0
Number of Namespaces:               1
Namespace 1 Size/Capacity:          214,748,364,800 [214 GB]
Namespace 1 Formatted LBA Size:     512
Local Time is:                      Wed Mar 13 04:21:53 2024 UTC
Firmware Updates (0x03):            1 Slot, Slot 1 R/O
Maximum Data Transfer Size:         64 Pages
Warning  Comp. Temp. Threshold:     70 Celsius
Namespace 1 Features (0x12):        NA_Fields *Other*

Supported Power States
St Op     Max   Active     Idle   RL RT WL WT  Ent_Lat  Ex_Lat
 0 +     0.01W       -        -    0  0  0  0  1000000 1000000

Supported LBA Sizes (NSID 0x1)
Id Fmt  Data  Metadt  Rel_Perf
 0 +     512       0         0

=== START OF SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

SMART/Health Information (NVMe Log 0x02)
Critical Warning:                   0x00
Temperature:                        -
Available Spare:                    0%
Available Spare Threshold:          0%
Percentage Used:                    0%
Data Units Read:                    0
Data Units Written:                 0
Host Read Commands:                 0
Host Write Commands:                0
Controller Busy Time:               0
Power Cycles:                       0
Power On Hours:                     0
Unsafe Shutdowns:                   0
Media and Data Integrity Errors:    0
Error Information Log Entries:      0
Warning  Comp. Temperature Time:    0

Error Information (NVMe Log 0x01, max 64 entries)
No Errors Logged
```

## 权限检查

```bash
ll /var/lib/ceph/6b6d577c-b035-11ee-8b7f-e51965640d97/osd.0
```

查看 ceph osd 0 所在文件夹权限发现无问题

```plaintext
lrwxrwxrwx 1 167 167  111 Mar 13 04:16 block -> /dev/mapper/ceph--2ee73f01--0a14--4ea0--a33d--59977dd4e642-osd--block--8e734088--2ddc--4a50--8e04--9d4570e7ae88
-rw------- 1 167 167   37 Mar 13 04:16 ceph_fsid
-rw------- 1 167 167  259 Mar 13 04:16 config
-rw------- 1 167 167   37 Mar 13 04:16 fsid
-rw------- 1 167 167   55 Mar 13 04:16 keyring
-rw------- 1 167 167    6 Mar 13 04:16 ready
-rw------- 1 167 167    3 Mar 13 04:16 require_osd_release
-rw------- 1 167 167   10 Mar 13 04:16 type
-rw------- 1 167 167   38 Mar 13 04:16 unit.configured
-rw------- 1 167 167   48 Mar 13 04:16 unit.created
-rw------- 1 167 167   90 Mar 13 04:16 unit.image
-rw------- 1 167 167  361 Mar 13 04:16 unit.meta
-rw------- 1 167 167 1.7K Mar 13 04:16 unit.poststop
-rw------- 1 167 167 2.8K Mar 13 04:16 unit.run
-rw------- 1 167 167  330 Mar 13 04:16 unit.stop
-rw------- 1 167 167    2 Mar 13 04:16 whoami
```

判断问题应该出在 Ceph 的文件系统损坏了，需要重建文件系统

# 恢复过程

## 卸载 osd

```bash
ceph osd out osd.0
ceph osd purge 0 --yes-i-really-mean-it
ceph osd crush remove osd.0
ceph auth rm osd。0
ceph osd rm osd.0
```

## 清理磁盘

```bash
sgdisk --zap-all /dev/nvme1n1
wipefs -a -f /dev/nvme1n1
blkdiscard /dev/nvme1n1
partprobe /dev/nvme1n1
```

找到对应磁盘的卷组删除

```bash
ls /dev/mapper/ceph-*
rm -rf /dev/mapper/ceph--52ae98df--3053--40af--86c4--def1ed9b9e68-osd--block--f09b0512--e73a--473b--8316--ccfe092c142d
```

```bash
ceph-volume lvm zap /dev/nvme1n1 --destroy
```

报错，表明当前磁盘正在被占用，需要解除占用后，再格式化

```plaintext
--> Zapping: /dev/nvme1n1
 stderr: wipefs: error: /dev/nvme1n1: probing initialization failed: Device or resource busy
--> failed to wipefs device, will try again to workaround probable race condition
 stderr: wipefs: error: /dev/nvme1n1: probing initialization failed: Device or resource busy
--> failed to wipefs device, will try again to workaround probable race condition
 stderr: wipefs: error: /dev/nvme1n1: probing initialization failed: Device or resource busy
--> failed to wipefs device, will try again to workaround probable race condition
 stderr: wipefs: error: /dev/nvme1n1: probing initialization failed: Device or resource busy
--> failed to wipefs device, will try again to workaround probable race condition
 stderr: wipefs: error: /dev/nvme1n1: probing initialization failed: Device or resource busy
--> failed to wipefs device, will try again to workaround probable race condition
```

查看是否有进程占用，如有则需要清理进程

```bash
lsof /dev/nvme1n1
fuser /dev/nvme1n1
```

查看是否有设备映射，需要删除设备映射

```bash
dmsetup ls
dmsetup remove ceph--52ae98df--3053--40af--86c4--def1ed9b9e68-osd--block--f09b0512--e73a--473b--8316--ccfe092c142d
```

## 重新添加osd

```bash
ceph orch daemon add osd ceph1:/dev/nvme1n1
```

# 检查健康情况

重新检查 ceph 集群状态，发现已经 osd 已经正常，需要等待 pg 恢复

```bash
ceph -s
ceph osd tree
ceph osd status
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/ceph-osd-down/  

