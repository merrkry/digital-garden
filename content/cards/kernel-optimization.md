---
title: 代理服务器调优
tags:
  - 科学上网
date: 2024-01-08
---

缝合了一些常见的优化方案，理论上对代理服务器的延迟和吞吐量都有提升。此外，还开启了端口转发的内核支持。

## 安装 [XanMod 内核](https://xanmod.org/)

Xanmod 是一个流行的非官方 Linux 内核。在网络方面，它集成了 [BBRv3](https://github.com/google/bbr/tree/v3) 和 [Cloudflare 的内核补丁](https://github.com/cloudflare/linux/blob/master/patches/0014-add-a-sysctl-to-enable-disable-tcp_collapse-logic.patch)。（需要配合内核参数，见下文）

以下内容适用于 Debian。

首先执行

```shell
awk -f <(wget -O - https://dl.xanmod.org/check_x86-64_psabi.sh)
```

确认你的 CPU 架构，下面以 `x64v3` 为例（相应地修改你要安装的包名）。

```shell
wget -qO - https://dl.xanmod.org/archive.key | sudo gpg --dearmor -o /usr/share/keyrings/xanmod-archive-keyring.gpg
```

```
echo 'deb [signed-by=/usr/share/keyrings/xanmod-archive-keyring.gpg] http://deb.xanmod.org releases main' | sudo tee /etc/apt/sources.list.d/xanmod-release.list
```

```
sudo apt update && sudo apt install linux-xanmod-x64v3
```

安装完成后重启机器，输入 `uname -r` 检查当前安装的内核。

## 修改内核参数

把如下内容添加到 `/etc/sysctl.conf` 末尾：

> [!warning] 关于 TCP Fast Open
> 
> 关于 TFO 有一些争议，参见 [naiveproxy Wiki](https://github.com/klzgrad/naiveproxy/wiki/Performance-Tuning) 和 [Surge Knowledge Base](https://kb.nssurge.com/surge-knowledge-base/v/zh/technotes/tfo)。
> 
> 上面的配置遵从 ArchWiki 的建议，完整开启了 TFO。如果希望禁用，请设置 `net.ipv4.tcp_fastopen = 0`。

```shell
# BBR congestion control
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# Cloudflare's optimization, kernel patch required
net.ipv4.tcp_rmem = 8192 262144 536870912
net.ipv4.tcp_wmem = 8192 262144 536870912 # Cloudflare's original value: 4096 16384 536870912
net.ipv4.tcp_adv_win_scale = -2
net.ipv4.tcp_collapse_max_bytes = 6291456
net.ipv4.tcp_notsent_lowat = 131072

# Allow forwarding
net.ipv4.conf.all.route_localnet = 1
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.default.forwarding = 1

# Recommended by Arch Linux wiki
net.core.netdev_max_backlog = 16384
net.core.somaxconn = 8192
net.core.rmem_default = 1048576
net.core.rmem_max = 16777216
net.core.wmem_default = 1048576
net.core.wmem_max = 16777216
net.core.optmem_max = 65536
# duplicate of cloudflare's optimizations
# net.ipv4.tcp_rmem = 4096 1048576 2097152 
# net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.udp_rmem_min = 8192
net.ipv4.udp_wmem_min = 8192
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
net.ipv4.tcp_mtu_probing = 1
# net.ipv4.tcp_timestamps = 0 # may cause a security risk
```

## 参考资料

- [Linux 网络优化 - NNR](https://nnr.moe/knowledge/Linux%E7%BD%91%E7%BB%9C%E4%BC%98%E5%8C%96)
- [sysctl - ArchWiki](https://wiki.archlinux.org/title/sysctl#Improving_performance)
- [Optimizing TCP for high WAN throughput while preserving low latency - Cloudflare](https://blog.cloudflare.com/optimizing-tcp-for-high-throughput-and-low-latency)
