---
title: 在 pve lxc 容器中使用 pytorch cuda
date: 2023-03-20
draft: false
tags: ["pytorch"]
categories: ["工具"]
---

处理器：AMD 5600G

显卡：Tesla P40

主板：ASUS TUF B450M-PLUS GAMING

PVE版本：7.2-3

LXC容器镜像：debian-11-standard_11.6-1_amd64

## 显卡安装

P40 显卡没有视频输出接口，在 BISO 中检查 `Primary Display` 选项，使用 IGFX 选项，避免无法显示界面。

P40 显卡有 24G 显存，需提前在 BIOS 开启 `Above 4G Decoding` （华硕的中文界面选项是“大于4G地址空间解码”）

## 宿主机安装驱动

安装驱动需编译内核模块，所以要先安装用于构建的软件包。

编辑 `/etc/apt/sources.list.d/pve-enterprise.list` 注释 pve 企业订阅源，添加非订阅源

``` txt
# deb https://enterprise.proxmox.com/debian/pve bullseye pve-enterprise

deb http://download.proxmox.com/debian/pve bullseye pve-no-subscription
```

执行 `apt update && apt install build-essential pve-headers-$(uname -r)`

<!--more-->

到 <https://www.nvidia.com/download/index.aspx> 下载显卡对应的驱动

执行 `chmod +x NVIDIA-Linux-x86_64-460.106.00.run && ./NVIDIA-Linux-x86_64-460.106.00.run`

在安装过程中，会自动生成禁用内核自带的 nouveau 驱动的配置文件，选 yes 即可，如果手误选了 no ，可以添加下列配置到 `/etc/modprobe.d/nvidia-installer-disable-nouveau.conf` 文件

```txt
# generated by nvidia-installer
blacklist nouveau
options nouveau modeset=0
```

驱动安装完毕后，添加一个 rules 配置，用于创建后面所需的设备

```bash
# /etc/udev/rules.d/70-nvidia.rules
# Create /nvidia0, /dev/nvidia1 … and /nvidiactl when nvidia module is loaded
KERNEL=="nvidia", RUN+="/bin/bash -c '/usr/bin/nvidia-smi -L && /bin/chmod 666 /dev/nvidia*'"
# Create the CUDA node when nvidia_uvm CUDA module is loaded
KERNEL=="nvidia_uvm", RUN+="/bin/bash -c '/usr/bin/nvidia-modprobe -c0 -u && /bin/chmod 0666 /dev/nvidia-uvm*'"
```

重启宿主机主机，以便驱动和rules生效，重启后执行 `nvidia-smi` 测试驱动是否正常工作。

## lxc 容器配置

在宿主机执行 `ls -l /dev/nvidia*` 查看设备号

```txt
crw-rw-rw- 1 root root 195,   0 Mar 19 21:09 /dev/nvidia0
crw-rw-rw- 1 root root 195, 255 Mar 19 21:09 /dev/nvidiactl
crw-rw-rw- 1 root root 506,   0 Mar 19 21:09 /dev/nvidia-uvm
crw-rw-rw- 1 root root 506,   1 Mar 19 21:09 /dev/nvidia-uvm-tools

/dev/nvidia-caps:
total 0
cr-------- 1 root root 235, 1 Mar 19 21:09 nvidia-cap1
cr--r--r-- 1 root root 235, 2 Mar 19 21:09 nvidia-cap2
```

其中 195、506、235 就是设备号，需要把设备号追加到 lxc 容器的配置文件 `/etc/pve/lxc/103.conf` （103 是容器 id，要改成你自己的容器 id） 并添加设备挂载配置项，如：

```txt
lxc.cgroup.devices.allow: c 195:* rwm
lxc.cgroup.devices.allow: c 506:* rwm
lxc.cgroup.devices.allow: c 235:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file
```

启动容器并在容器中再次安装驱动 `./NVIDIA-Linux-x86_64-460.106.00.run --no-kernel-module` 由于容器和宿主机是公用内核的，不需要也无法安装内核模块，所以在安装时加 `--no-kernel-module` 参数，安装完毕后在容器中执行 `nvidia-smi` 测试驱动是否正常

## lxc 容器安装 pytorch

到 <https://docs.conda.io/en/latest/miniconda.html> 下载最新版 miniconda 并在容器中安装 `chmod +x Miniconda3-py39_23.1.0-1-Linux-x86_64.sh && ./Miniconda3-py39_23.1.0-1-Linux-x86_64.sh`。

conda 安装完成后执行 `conda install pytorch torchvision torchaudio pytorch-cuda=11.7 -c pytorch -c nvidia` 安装 pytorch 最新版本。

**可提前配置 <https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/> 镜像加速 conda 下载**

pytorch 安装完毕后，执行 `python -c 'import torch; print(torch.cuda.is_available())'` 测试，如果打印 True 表示安装成功并且支持 cuda 加速。

如果输出为 False ，可能是显卡驱动版本太老，不兼容最新版本的 pytorch ，在 <https://pytorch.org/get-started/previous-versions/> 查找旧版本 pytorch 安装方法。