---
title: SSH 常用功能
date: 2022-06-19T00:30:16+08:00
draft: false
tags: ["网络"]
categories: ["工具"]
---

SSH 是 Linux 下最常见的远程登录服务，通过非对称加密，可以在不安全的网络环境中建立安全的传输环境。本文介绍ssh免密登录、
跳转登录、端口转发、代理隧道等应用。

## 安装

deepin 系统默认集成 OpenSSH，无需手动安装。但出于安全考虑，默认未启用 ssh 服务，使用 `sudo systemctl start ssh` 启用 ssh 服务。

<!--more-->

> ！警告：网络上存在大量 ssh 扫描，如你的账号使用弱密码，请关闭 ssh 密码认证模式。

## 登录

### 密码登录

使用`ssh username@hostname`可使用指定用户登录到指定的主机，如用户名和你当前使用的账户用户名相同，可省略用户名。如`ssh localhost`可登录到本机，`ssh deepin@x.x.x.x`在 x.x.x.x 主机登录 deepin 账户。

默认情况下，登录需要校验用户密码，首次登录到主机还需确认主机 ssh 指纹，避免中间人攻击。如果主机未进行操作，却发生 ssh 指纹变化，ssh 登录时会返回警告 WARNING: POSSIBLE DNS SPOOFING DETECTED!，请警惕中间人攻击，如果你刚重装了主机的操作系统，可根据 ssh 警告提示，使用`ssh-keygen -f`移除之前的主机指纹记录，再进行登录。

### 公钥免密登录

使用`ssh-copy-id localhost`可自动配置免密登录，再次使用 ssh 登录时，则无需输入密码。如 ssh-copy-id 返回 No identities found 错误，使用 ssh-keygen 先生成你的公钥（ssh-keygen 一路回车即可）。

你也可手动复制你的公钥（默认位置在`~/.ssh/id_rsa.pub`）到主机的`~/.ssh/authorized_keys`文件中，以配置免密登录。

在配置免密登录后，可关闭 ssh 的密码登录以避免密码暴力破解。使用命令`echo PasswordAuthentication no | sudo tee -a /etc/ssh/sshd_config`
修改配置，需使用命令 `sudo systemctl start ssh` 重启 ssh 服务才会生效。

> ！警告：请确定配置免密成功再关闭密码登录，否则你的主机可能无法再登录使用。

### 免密跳转登录

如果你有多台主机，可能会需要进行跳转登录，即先登录 A 主机，再通过 A 主机登录 B 主机，因为你可能无法直接访问 B 主机。
有两种方式实现跳转登录：`ssh -A`和`ssh -J`，使用`ssh -A hostA`免密登录到主机 A 后，可继续在命令行使用`ssh hostB`免密登录到主机 B。`ssh -J hostA hostB`则可一步到位的通过 hostA 跳转登录到 hostB。

## 端口转发

### 本地转发

ssh 可以将远程端口转发到本地，下面给几个常用例子

- 转发 本地的 80 端口到本地的 8080 端口，可用于临时 TCP 端口映射

  `ssh -L 8080:localhost:80 localhost`

- 转发 hostA 主机的 3306 端口到本地的 13306 端口，通常用于本地调试时使用远程服务。

  `ssh -L 127.0.0.1:3306:127.0.0.1:3306 hostA`

  > 出于安全考虑，转饭时最好在前面加上 127.0.0.1，这样转发的端口绑定到本地 127.0.0.1 地址上，避免本地局域网扫描端口。

- 通过 hostA 主机，转发 192.192.0.1 主机的 80 端口到本地的 8080 端口，通常用于远程访问网关、路由器等。

  `ssh -L 127.0.0.1:8080:192.192.0.1:80 hostA`

### 远程转发

ssh 可以将本地端口转发到远程，下面给几个常用例子。

- 转发本地的 8000 端口到本地的 8080 端口，可用于临时 TCP 端口映射

  `ssh -R 8080:localhost:8000 localhost`

- 转发本地的 8080 端口到 hostA 的 8000 端口，通常用于在本地无公网 IP 时，映射端口

  `ssh -R 8000:localhost:8080 hostA`

  > 出于安全考虑，ssh 远程转发，默认绑定在 127.0.0.1，如果需要外部通过 hostA 访问转发的端口，可修改远程主机的 sshd 配置文件，开启 GatewayPorts 选项

- 转发 192.168.0.1 的 80 端口到 hostA 的 8000 端口，通常用于内网机器端口映射

  `ssh -R 8000:192.168.0.1:80 hostA`

## Socks5 代理

ssh 可以在本地建立一个 socks 代理，通过代理，使用 hostA 访问任何 TCP 地址。

在本地 7000 端口建立 socks 代理 `ssh -D 127.0.0.1:7000 hostA`， 其他软件可通过 socks 代理访问网络，如果需要在其他主机上使用你本地的 socks 代理，使用 `ssh -D 7000 hostA`。

## TUN 隧道

无论是端口转发还是 Socks5 代理，ssh 只支持 tcp 协议，如果你需要转发 udp 协议，可以尝试使用 ssh 的 tun 隧道，tun 隧道可以在主机之间组建虚拟网络。

> 使用 ssh 的 tun 功能，需要修改远程主机的 ssh 服务配置文件，开启 PermitTunnel 选项。

虽然 ssh 会尝试自动创建 tun 设备，但 tun 设备需要 root 用户才能创建，一般不推荐使用 root 账户登录，所以在最开始，我们来手动在本地和远程主机分别创建 tun 设备。

- 在本地主机创建 tun 设备并设置 IP 地址 192.168.111.2

  ```bash
  sudo ip tuntap add dev tun0 mode tun
  sudo ip link set tun0 up
  sudo ip address add 192.168.111.2 peer 192.168.111.3 dev tun0
  ```

- 在远程主机创建 tun 设备并设置 IP 地址 192.168.111.3

  ```bash
  sudo ip tuntap add dev tun0 mode tun
  sudo ip link set tun0 up
  sudo ip address add 192.168.111.3 peer 192.168.111.2 dev tun0
  ```

  创建设备之后，使用`ssh -w 0:0 hostA` 即可通过 ssh 连接 tun 设备，本地可使用 192.168.111.3 地址，向远程发送任何 TCP/UDP 协议的数据，在远程主机也可使用 192.168.111.2 地址，向本地发送数据，不再局限于某个端口或某个协议。
