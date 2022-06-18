---
title: "Bcache in Deepin"
date: 2022-06-19T00:09:58+08:00
draft: false
tags: ["存储"]
categories: ["工具"]
---

# Bcache 使用

## Bcache 是什么

Bcache 是 Linux 下的一个块缓存内核模块，可用于使用单个固态硬盘为一个或多个机械硬盘加速，也可用于使用本地磁盘为网络磁盘的加速。

Bcache 有比较灵活的缓存模式，支持安全读写缓存(writethrough)、高性能读写缓存(writeback)、只读缓存(writearound)、停用缓存(none)等模式，并且可在线动态调整。

Bcache 还会自动识别顺序写入，当发现正在进行顺序写入时跳过缓存层，以减少缓存设备(固态硬盘)的损耗，延长固态硬盘的寿命。

## 快速开始

> ！警告：使用 Bcache 需要格式化相应设备，需谨慎操作。

### 安装

deepin 仓库已存在 Bcache 1.0.8 版本，可直接在终端使用 `sudo apt install bcache-tools` 命令安装。

### 格式化

安装完成后，可使用 `sudo make-cache -C /dev/nvmeX -B /dev/sdX` 格式化缓存设备(nvmeX)和后端设备(sdX)，并自动关联两个设备。

> 可能出现的错误
>
> - _Device or resource busy_ 设备被占用，用 df 命令查看设备是不是已经被挂载使用，用 umount 取消挂载。
> - _Device /dev/sdX already has a non-bcache superblock_ 设备已存在文件系统，为避免文件丢失，请在**备份文件**后，使用 `sudo wipefs -a /dev/sdX` 清除设备。

在顺利格式化后，会出现一个 /dev/bcache0 设备，大小与后端设备一致，可以当作普通的块设备使用。
例如使用 `sudo mkfs.btrfs /dev/bcache0` 格式化为 btrfs 文件系统格式，使用 `sudo mount /deb/bcache0 /mnt` 挂载。

### 调整缓存模式

默认情况下，Bcache 使用安全读写缓存(writethrough)模式，在该模式下，数据落盘会同时写入到固态硬盘和机械硬盘上，以避免固态硬盘损坏导致数据丢失，但这样会影响写入速度。在无严格需求的情况下，可改为高性能读写(writeback)模式，该模式下数据会先写入到固态硬盘，再定时同步到机械硬盘。

查看当前缓存模式 `cat /sys/block/bcache0/bcache/cache_mode`

```bash
[writethrough] writeback writearound none
```

动态修改缓存模式 `echo writeback | sudo tee /sys/block/bcache0/bcache/cache_mode`。

> 修改实时时效

## 效果测试

### 使用 fio 进行随机读测试

```bash
sudo fio -filename=/dev/XXX -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=16k -size=5G -numjobs=30 -runtime=10 -group_reporting -name=mytest
```

- 固态硬盘速度

```bash
Run status group 0 (all jobs):
   READ: bw=433MiB/s (454MB/s), 433MiB/s-433MiB/s (454MB/s-454MB/s), io=4330MiB (4540MB), run=10002-10002msec
```

- 机械硬盘速度

```bash
Run status group 0 (all jobs):
   READ: bw=6664KiB/s (6824kB/s), 6664KiB/s-6664KiB/s (6824kB/s-6824kB/s), io=65.7MiB (68.9MB), run=10099-10099msec
```

- Bcache 速度

```bash
Run status group 0 (all jobs):
   READ: bw=20.8MiB/s (21.8MB/s), 20.8MiB/s-20.8MiB/s (21.8MB/s-21.8MB/s), io=211MiB (221MB), run=10140-10140msec
```

### 使用 fio 进行顺序读测试

```bash
sudo fio -filename=/dev/XXX -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=16k -size=5G -numjobs=30 -runtime=8 -group_reporting -name=mytest
```

- 固态硬盘速度

```bash
Run status group 0 (all jobs):
   READ: bw=442MiB/s (463MB/s), 442MiB/s-442MiB/s (463MB/s-463MB/s), io=3536MiB (3707MB), run=8008-8008msec
```

- 机械硬盘速度

```bash
Run status group 0 (all jobs):
   READ: bw=295MiB/s (309MB/s), 295MiB/s-295MiB/s (309MB/s-309MB/s), io=2357MiB (2472MB), run=8002-8002msec
```

- Bcache 速度

```bash
Run status group 0 (all jobs):
   READ: bw=349MiB/s (366MB/s), 349MiB/s-349MiB/s (366MB/s-366MB/s), io=2795MiB (2931MB), run=8002-8002msec
```

### 测试结论

从测试结果来看，Bcache 两倍的随机读性能（和固态硬盘的性能成正比），在实际使用中体现为 deepin 在输入用户密码后更快的显示桌面，日常软件的能更快的启动。

## 常用操作

- 格式化缓存设备

  `sudo make-bcache -C /dev/nvmeX`

- 格式化后端设备

  `sudo make-bcache -B /dev/sdX`

- 查看设备信息

  `sudo bcache-super-show /dev/XXXX`

- 附加缓存设备到后端设备

  `echo $(cset.uuid) | sudo tee /sys/block/bcache0/bcache/attach`

  > cset.uuid 使用 bcache-super-show 查看
  >
  > 一个缓存设备可附加到多个后端设备

- 从后端设备分离缓存设备

  `echo $(cset.uuid) | sudo tee /sys/block/bcache0/bcache/detach`

- 停用缓存设备

  `echo 1 | sudo tee /sys/fs/bcache/$(cset.uuid)/unregister`

  > 为避免文件丢失，缓存设备应该先分离再停用。

- 停用后端设备

  `echo 1 | sudo tee /sys/block/bcache0/bcache/stop`
