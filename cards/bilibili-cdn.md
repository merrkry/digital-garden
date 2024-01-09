---
title: 优化 CDN 选取，解决 B 站卡顿问题
tags: 
date: 2024-01-09
---

b 站的技术力低得让人难以置信。本地电信千兆网络，在晚高峰最低分辨率都有可能卡顿甚至看不了……

B 站的 CDN 有四种：

- PCDN (Peer-to-Peer Content Delivery Network)
- MCDN (Mobile Content Delivery Network)
- 自建 CDN
- 公有云 CDN

PCDN 顾名思义，依赖于其他用户来传输内容，在对速度和延迟有要求的场景下显然完全不可靠。

MCDN 搜到的全是 CDN 提供商的广告，没有找到靠谱的介绍。[这篇文章](https://martechseries.com/mts-insights/guest-authors/mobile-cdn-what-is-it-and-why-is-it-essential-for-mobile-apps/) 提到 MCDN 是优化「最后一公里」（即 CDN 到用户这一段）的解决方案，且专注于移动设备（这一点尤其迷惑，没有任何一篇文章说清楚到底怎么对移动设备优化的）。[据称 B 站的 CDN 和京东云有关](https://www.shawnleetttt.cyou/posts/457eb4a4/)。实践上来讲，B 站尤其容易给冷门内容分配 MCDN，而连接质量往往不尽人意。

后两者的连接质量相对稳定，适合使用。

## 禁用 PCDN 和 MCDN

使用代理软件的分流规则，屏蔽这两种 CDN。以 Clash 为例：

```
- DOMAIN-SUFFIX, v1d.szbdyd.com, REJECT
- DOMAIN-SUFFIX, mcdn.bilivideo.cn, REJECT
```

部分自建 CDN 和公有云 CDN 连接质量也不佳。理论上也可以进行优选（完整列表见 [CDN 信息 | 录播姬](https://rec.danmuji.org/dev/cdn-info/)）。但是测试、筛选和屏蔽都过于繁琐，不推荐折腾。

## 使用运营商 DNS

为了避免运营商的 DNS 污染，配置代理软件时我们一般会选择使用污染程度相对低的公共 DNS。但国内公共 DNS 对 ECS（也即 EDNS，把客户端 IP 段暴露给权威 DNS 服务器以获得正确的服务器分配，见 [EDNS Client Subnet 协议简介](https://taoshu.in/dns/edns-client-subnet.html)）的支持往往并不完整，CDN 分配效果并不理想。运营商在各级局域网都部署了 DNS，能够实现较好的**就近接入**。

推荐使用 DNS 分流规则，为国内网站使用系统 DNS（默认情况下会通过 DHCP 获取 DNS，后者默认为运营商 DNS）。

```
"geosite:cn": system
```

注意 CMFA 没有特权，请使用 `dhcp://system`。（[\[Bug\] CMFA无法使用\`system\`来获取本机DNS · Issue #927 · MetaCubeX/mihomo · GitHub](https://github.com/MetaCubeX/mihomo/issues/927#issuecomment-1862420914)）
