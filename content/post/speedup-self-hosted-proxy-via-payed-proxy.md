---
title: "使用机场加速自建代理"
description: 用 clash 做代理客户端，使用机场加速自建代理，兼顾安全与高速
date: 2022-06-25T11:14:47+08:00
image: https://raw.githubusercontent.com/Dreamacro/clash/master/docs/logo.png
hidden: false
comments: true
tags: [proxy, clash]
categories: [Network]
draft: false
---

自建代理与机场比较
||自建代理|机场|
|---|---|---|
速度|慢|快
安全性|高|无

有没有兼顾安全性和速度的方案？

## 最终方案
```
+-----------+      +---------------+
|           |      |               |
| 墙内机器  +--①--->  机场国内中转 |
|           |      |               |
+-----------+      +-------+-------+
                           |
                           |②
+------------+     +-------v-------+
|            |     |               |
| Cloudflare <--③--+ 机场国外节点  |
|            |     |               |
+------+-----+     +---------------+
       |
       |④
+------v-----+     +---------------+
|            |     |               |
|  自建梯子  +--⑤-->  目标网站     |
|            |     |               |
+------------+     +---------------+
```



### 部署要点

1. 自建梯子用 tsl+websocket 协议，各种工具都可以，ss+v2ray-plugin、v2ray、gost 等都可以。这一步目的是为了能用 Cloudflare 的 CDN。
2. 需要有自己的域名，把域名接入 Cloudflare。
3. 在本地用 clash 控制流量走向

## 方案优势

1. 国内运营商在链路①的位置只能看到你访问了机场的国内 IP
2. 机场在链路②③只能看到你在用 tls 流量访问 Cloudflare 节点，目标网站就是你接入 Cloudflare 的域名。
3. **只要你不用代理访问国内网站，运营商和机场都不知道你的自建梯子的出口 IP**
4. 和直连自建梯子比，大幅提升速度和稳定性；和单独使用机场相比，机场无法知道你的访问信息，安全性很大提升



## 方案实施

1. 各代理软件都有详细文档，如何开启tsl+websocket 协议，此处略
2. 域名接入 cloudflare， 略
3. 购买机场，略
4. clash 配置

### Clash 简介

[Clash](https://github.com/Dreamacro/clash)是一款用 Go 开发的多平台的代理工具，使用强大的策略组来管理节点;可以自动选择节点，不同的网站，在同一时间，可以使用不同的节点去访问。


我们用 Clash 强大的策略组功能来套娃。

官方文档有示例 https://github.com/Dreamacro/clash/wiki/configuration
> relay chains the proxies. proxies shall not contain a relay. No UDP support.
> Traffic: clash <-> http <-> vmess <-> ss1 <-> ss2 <-> Internet

### 配置文件
```yaml
mixed-port: 7890
mode: rule

proxies:
  # 自建 ss
  - name: "my-vps-ss"
    type: ss
    server: mydomain.xx.cc # 这里也可以是 Cloudflare 的 IP
    port: 443
    cipher: chacha20-ietf-poly1305
    password: "password"
    plugin: v2ray-plugin
    plugin-opts:
      mode: websocket
      tls: true
      skip-cert-verify: true
      host: mydomain.xx.cc

  # 机场 ss1
  - name: "jichang-ss-hk"
    type: ss
    server: jichang.abc.com
    port: 12345
    cipher: chacha20-ietf-poly1305
    password: "password"
    plugin: obfs
    plugin-opts:
      mode: http
      host: bing.com

  # 机场 ss2
  - name: "jichang-ss-tw"
    type: ss
    server: jichang.abc.com
    port: 12346
    cipher: chacha20-ietf-poly1305
    password: "password"
    plugin: obfs
    plugin-opts:
      mode: http
      host: bing.com

  # 机场 v2ray
  - name: "jichang-v2ray-hk"
    type: vmess
    server: jichang.abc.com
    port: 12347
    cipher: auto
    uuid: xxxx-xxxx-xxxx-xxxx
    alterId: 64

proxy-groups:
  # 先把所有机场结点放在一个组
  - name: "机场"
    # 自动选择延时最小的结点
    type: url-test
    proxies:
      - jichang-ss-hk
      - jichang-ss-tw
      - jichang-v2ray-hk
    url: 'http://www.gstatic.com/generate_204'
    interval: 300

  # 创建一个 type 是 relay 的组，流量按 proxies 里的顺序，从上往下走
  - name: "myproxy"
    # 自动选择延时最小的结点
    type: url-test
    proxies:
      - 机场
      - my-vps-ss

rules:
  - IP-CIDR,127.0.0.0/8,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,myproxy

```

至此，配置完成，7980 端口同时提供 http 和 socks4、socks5 代理。

分流策略是：
- 国内 IP 直连
- 国外 IP 走代理

### 数据流：
1. 本机请求 -> clash，clash 按 relay 组的 proxies 顺序，从下往上加密，本配置中，先用自建的加密方式加密请求，再用机场的加密方式加密第二次
2. clash -> 机场，机场解开自己的加密，看到目标是一个 Cloudflare 上的网站
3. 机场 -> Cloudflare, 用标准的 https 方式转发收到的数据包
3. Cloudflare -> vps，vps 收到 https 包后，先用域名的私钥解密数据，得到加密的 ss 数据，再用 ss 解密得到真正的目标。

## 更古老的套娃方案
在遇到 clash 之前，一直在 openwrt 上套娃，使用的是 shadowsocks + iptables-mod-tproxy
### 环境准备
```bash
opkg install shadowsocks-libev
```
### 操作方法
1. openwrt 上正常配置自建梯子，能正常使用后进入第2步
2. 用 ss-redir 启动机场的 ss
  ```bash
  ss-redir -c jichang.json -l 12346
  ```
3. iptables 把自建的流量重定向到机场。假设自建的 IP 是 1.1.1.1
  ```bash
  iptables -t nat -I OUTPUT -p tcp -d 1.1.1.1 -j REDIRECT --to-ports 12346
  ```

  **机场的 ss-redir 需要后启动，shadowsocks 默认策略是插入 OUTPUT 链首位，后启动的在首位**

## 参考链接
- [使用Clash工具科学上网](https://wanchuan.top/e9430b8345c049f8afce602db5f5f773)
- [clash 官方配置文档](https://github.com/Dreamacro/clash/wiki/configuration)
- [v2ay 配置透明代理规则](https://guide.v2fly.org/app/tproxy.html#%E9%85%8D%E7%BD%AE%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E8%A7%84%E5%88%99)
- [aa65535 维护的 shadowsocks for openwrt](http://openwrt-dist.sourceforge.net/)
