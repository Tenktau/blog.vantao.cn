---
title: 在 PVE 上用 USB 无线网卡为虚拟机提供网络访问
description: 系统性问题的排雷上，LLM 也并非 100% 可靠…
date: 2026-03-08 +0800
categories: [Development, Software]
tags: [PVE, 虚拟机, 操作系统]
---

> 声明：本文由 Claude 采样于与作者的对话而自主生成，并对隐私信息作了模糊处理，参考时建议仔细甄别。

## 一、需求背景

家用台式机上安装了 Proxmox VE（PVE）作为虚拟化平台，计划在其中运行 Ubuntu 虚拟机部署自托管服务。然而机器没有有线网口，只有一块 USB 无线网卡。目标是：

- PVE 宿主机通过无线网卡正常上网
- 虚拟机能借助宿主机的网络访问互联网
- 局域网内其他设备可以通过端口转发访问虚拟机内部署的服务

## 二、踩坑记录

### 2.1 无线网卡无法直接桥接给虚拟机

最直观的想法是把无线网卡 `wlx04f9f83f0000` 直接加入 PVE 的虚拟网桥 vmbr0，然后让虚拟机通过桥接获取路由器 IP。但这条路行不通：

> WiFi 协议的工作机制决定了无线 AP 只会转发已关联 MAC 地址的数据帧。虚拟机的虚拟网卡 MAC 地址对 AP 来说是陌生的，数据帧会被直接丢弃。

### 2.2 建虚拟网桥后 PVE 宿主机断网

尝试在 `/etc/network/interfaces` 中添加 vmbr0 配置时，由于配置了 `gateway`，与无线网卡通过 DHCP 已获得的默认路由产生冲突，导致 PVE 宿主机网络中断。

原始配置文件片段（问题所在已被注释）：

```
#auto vmbr0
#iface vmbr0 inet static
#       address 192.168.1.220/24
#       gateway 192.168.1.1        ← 与 wlan 的路由冲突
#       bridge-ports none
#       ...
```

### 2.3 docker-compose 文件损坏

虚拟机联网后，在网络不稳定的情况下下载 `docker-compose` 二进制文件时，实际保存的是服务器返回的 HTML 错误页，导致执行时报错：

```
/usr/local/bin/docker-compose: 行 1: `<!DOCTYPE html>'
```

### 2.4 apt 源中 docker-compose 版本与 Python 3.12 不兼容

通过系统源安装的 `docker-compose 1.29.2` 是 Python 实现的旧版本，而 Ubuntu 24.04 内置 Python 3.12 已移除 `distutils` 模块，直接导致运行崩溃：

```
ModuleNotFoundError: No module named 'distutils'
```

### 2.5 Docker Hub 国内访问超时

在国内环境直接拉取 Docker 镜像时，`registry-1.docker.io` 连接超时，安装脚本无法下载所需镜像。

## 三、解决方案

### 3.1 PVE 宿主机：添加纯虚拟内网桥 + NAT

在 `/etc/network/interfaces` 中新增一个**不绑定任何物理网卡、不设网关**的虚拟网桥 vmbr1，通过 iptables 做 NAT 将流量转发至无线网卡：

```
auto vmbr1
iface vmbr1 inet static
        address 192.168.100.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up   iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o wlx04f9f83f0000 -j MASQUERADE
        post-up   iptables -A FORWARD -i vmbr1 -o wlx04f9f83f0000 -j ACCEPT
        post-up   iptables -A FORWARD -i wlx04f9f83f0000 -o vmbr1 -m state --state RELATED,ESTABLISHED -j ACCEPT
        post-down iptables -t nat -D POSTROUTING -s 192.168.100.0/24 -o wlx04f9f83f0000 -j MASQUERADE
        post-down iptables -D FORWARD -i vmbr1 -o wlx04f9f83f0000 -j ACCEPT
        post-down iptables -D FORWARD -i wlx04f9f83f0000 -o vmbr1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

执行 `ifup vmbr1` 使其生效，无需重启，也不影响无线网卡的现有连接。

### 3.2 PVE 控制台：将虚拟机网卡挂载到 vmbr1

在 PVE Web 界面中，进入虚拟机 → Hardware → Network Device，将 Bridge 改为 `vmbr1`。

### 3.3 Ubuntu 虚拟机：配置静态 IP（netplan）

编辑 `/etc/netplan/00-installer-config.yaml`：

```yaml
network:
  version: 2
  ethernets:
    ens16:
      addresses: [192.168.100.10/24]
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 114.114.114.114]
```

```bash
sudo netplan apply
```

### 3.4 修复 docker-compose

卸载旧的 Python 版本，安装 Go 实现的插件版并建立软链接：

```bash
sudo apt remove docker-compose -y
sudo apt install docker-compose-plugin -y
sudo ln -s /usr/libexec/docker/cli-plugins/docker-compose /usr/local/bin/docker-compose
docker-compose version
```

### 3.5 配置 Docker 国内镜像源

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://docker.1panel.live",
    "https://hub.rat.dev",
    "https://dockerpull.org"
  ]
}
EOF

sudo systemctl daemon-reload && sudo systemctl restart docker
```

### 3.6 局域网端口转发

虚拟机处于 NAT 内网，局域网其他设备无法直接访问。在 PVE 宿主机上添加 iptables 转发规则，将宿主机的指定端口映射到虚拟机：

```bash
# 以 3000 端口为例
iptables -t nat -A PREROUTING -p tcp --dport 3000 -j DNAT --to-destination 192.168.100.10:3000
iptables -A FORWARD -p tcp -d 192.168.100.10 --dport 3000 -j ACCEPT

# 持久化保存
apt install iptables-persistent -y && netfilter-persistent save
```

局域网设备通过 `http://[PVE宿主机IP]:3000` 访问服务。

## 四、实现效果

- PVE 宿主机通过 USB 无线网卡正常联网，与虚拟机网络配置完全解耦，互不干扰
- Ubuntu 虚拟机获得稳定的内网 IP（`192.168.100.10`），通过 NAT 正常访问互联网
- Docker 环境配置完整，镜像拉取正常，自托管服务可正常部署运行
- 局域网内其他设备可通过 PVE 宿主机 IP + 端口号访问虚拟机内的服务，无需额外网络配置

整套方案绕开了 WiFi 无法直接桥接的协议限制，以最小的配置改动实现了完整的虚拟机网络访问能力。