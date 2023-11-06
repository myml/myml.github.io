---
title: 最佳的家庭raid方案
date: 2023-11-06
draft: false
tags: ["存储", "raid"]
categories: ["工具"]
---

在信息时代每个家庭都有大量的数据，可能是照片、视频、密码或其他重要文件。因此数据的安全性和可靠性变得至关重要。

RAID（[独立磁盘冗余阵列](https://zh.wikipedia.org/wiki/RAID)）是一种常见的数据保护解决方案，我认为最佳的家庭 RAID 方案应该是 SnapRAID。SnapRAID 是一款开源软件，相较于传统 RAID 卡和软 RAID，SnapRAID 的主要优势在于其成本效益、灵活性和易用性。

<!--more-->

- SnapRAID 不要求所有硬盘容量相同
  
  阵列卡和软 raid 都要求硬盘容量一致，这在企业环境中相对容易实现，因为企业可以根据需要选择硬盘，以确保它们具有相同的容量和性能。然而，在个人家庭使用的情况下，因预算有限硬盘容量不一致可能更常见。

- SnapRAID 不劫持数据
  
  阵列卡和软 raid 在组建 raid 时需要格式化硬盘，除 raid1 外的其他 raid 类型通过算法将文件内容分散到多个硬盘，如果因意外导致 raid 无法恢复，将失去所有的数据，而使用 SnapRAID 不需要格式化，可以直接基于你现有的数据使用，在因多个硬盘损坏，导致无法恢复时，至少还可以访问未损坏硬盘中的文件。

- SnapRAID 不要求硬盘实时在线
  
  阵列卡和软 raid 在组建 raid 后，读写文件需要多个硬盘同时操作，而 SnapRAID 在平时读写文件仅操作数据所在的硬盘，从而降低了能耗并延长硬盘寿命，同时也减少了硬盘运行时的噪音。

在此外 SnapRAID 的设置和维护相对简单，无需专业知识。本文将详细介绍 SnapRAID 的使用方法，以确保你的数据得到可靠的保护。

<!--more-->

在 debian 上可以使用 `apt install snapraid` SnapRAID 的配置非常简单，

```txt
# 文件路径 /etc/snapraid.conf
data sda /mnt/share/sda/
data sdb /mnt/share/sdb/
data sdc /mnt/share/sdc/
data sdd /mnt/share/sdd/

parity /mnt/share/sde/snapraid.parity
content /mnt/share/sde/snapraid.content
content /mnt/share/sda/snapraid.content
```

配置文件有三个部分：data、parity、content。

- data 是数据盘，里面存储你的重要数据，你可以使用现有的，已存在数据的硬盘，snapraid 不会对现有数据做任何改动。

- parity 是校验盘，通过对 data 盘现有数据计算校验得出。一个校验盘可保证在任意一个 data 盘损坏后进行数据恢复，如果你觉得不够保险，可以使用 2-parity、3-parity 设置更多的校验盘。

- content 是索引文件的保存位置，主要记录数据盘的文件索引和校验数据索引，建议在数据盘和校验盘各存放一份。

在写好配置后，使用`snapraid sync`既可开始计算校验数据，初次运行会比较慢，第二次运行就会基于 content 和上次的校验做增量计算，速度就不会那么慢了。

当你的硬盘损坏或不小心格式化了数据盘，可以使用类似 `snapraid fix -m` 这样的命令来恢复数据盘到上次`sync`的状态。如果是校验盘损坏，也无需担心，再次运行`snapraid sync`重新生成校验既可。具体的操作可以参考 [SnapRAID 手册](https://www.snapraid.it/manual)，这里不再赘述。

当然 SnapRAID 也有自身的缺点：

- 不支持实时数据保护
  
  SnapRAID 需要定时执行`sync`来计算校验，如果在`sync`之前数据盘发生算坏，中间的数据变动是无法恢复的，这一点和传统 raid 相反，而正因为传统 raid 是实时生效的，反而无法恢复误删和误格式化。使用 SnapRAID 建议根据个人情况，设置定时任务执行`sync`。
- 不支持高可用

  阵列卡和软 raid 一般情况下是支持高可用的，在小部分硬盘损坏时，仍可正常读写，这在企业对服务可用性有要求的场景有用，对家庭场景来说没有太大的用处。
- 读写性能限制
  
  部分 raid 类型因为文件数据是分散到多个磁盘中，读写性能相应会有叠加的效果。而使用SnapRAID时，文件数据仅写入对应的数据盘，读写性能受限对应数据盘的速度。
- 扩展性受限

  软raid可扩展到数十甚至上百个硬盘，SnapRAID是基于文件的，在文件数量较多时会出现同步性能瓶颈，但在家庭场景是足够了。

尽管有这些限制，SnapRAID 仍然是我最推荐的raid方式，特别是在家庭用户和小型办公室用户之间，因为它提供了相对简单和成本效益的数据保护解决方案。并不是传统raid不好，而是SnapRAID更有性价比。