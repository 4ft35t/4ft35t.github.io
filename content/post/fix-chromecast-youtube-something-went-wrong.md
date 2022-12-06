---
title: "修复 Chromecast YouTube 投屏失败问题"
description: 路由器翻墙，ChromeCast 播放别的网站都没问题，唯独 YouTube 出错
date: 2022-07-29T01:38:00+08:00
image: https://www.right.com.cn/forum/data/attachment/forum/201803/08/102522uwh6r70ssbuh01r5.jpg
math: 
hidden: false
comments: true
tags: ["chromecast", "youtube", "openwrt", "iptables"]
categories: ["Network"]
draft: false
---

## 问题分析
软路由 OpenWrt 做网关，同时用 SS 透明代理所有国外 IP 的 TCP 连接，手机、电脑都无感翻墙。

Chromecast 在多个境外网站上投屏正常工作，Google Photos 投屏也正常，唯独 YouTube 无法投屏。

使用关键词 "chromecast youtube something went wrong" 没找到有用信息，把 ChromeCast 的语言改成中文后，用中文的报错信息一下就找到了，果然是天朝特有问题。Google 搜 `chromecast youtube "出问题了"`。


问题原因在于 Chromecast 不使用 DHCP 分配的 DNS 服务器，用 8.8.8.8 解析，会拿到污染的结果，导致失败。


## 解决方案
解决方法很简单，在网关上劫持 8.8.8.8 的 53 端口 UDP 包，重定向到网关的无污染 DNS。

```bash
iptables -t nat -I PREROUTING -p udp -d 8.8.8.8 --dport 53 -j REDIRECT --to-ports 53
```

OpenWrt 网页 - NetWork - Firewall - Custom Rules, 把上述命令贴进去，保存即可。

也可直接修改 `/etc/firewall.user` 然后 `/etc/init.d/firewall restart` 生效。

## 参考链接
- [Chromecast 如何配搭路由的 SS 看 youtube？](https://v2ex.com/t/436014)
- [极路由开发(Root)模式安装Shadowsocks, 以及Chromecast因DNS污染无法投射YouTube的解决方案](https://iso1.wordpress.com/2017/07/06/install-shadowsocks-on-geek-router-and-fix-chromecast-casting-youtube-issue/)
