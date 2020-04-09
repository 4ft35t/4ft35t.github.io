---
title: "解决无忧行长时间显示“正在连接电话服务器”问题"
date: 2020-04-09T21:48:15+08:00
tags: ["jegotrip", "IP", "pi-hole", "pihole"]
categories: ["Network"]
draft: false
---

无忧行(JegoTrip)APP是中国移动专门为出境游用户量身打造的APP。在境内也可以使用，移动号码可以在 APP 内接电话和收短信, 轻松实现一机双号。

## 具体症状
家里路由器上部署梯子, 国内 IP 直连，其余 IP 走梯子, 国内 IP 库用的 https://github.com/17mon/china_ip_list。连接家里 WIFI，无忧行 APP 启动后，或显示“正在连接电话服务器”, 持续 30 秒以上才能连接到服务器，之后才能收到短信；用 4G 则没有这个提示，启动  APP 瞬间就能收到短信。猜测是无忧行服务器的香港 IP 走梯子后比直连慢很多所致。
<!--more-->

## 排查 IP
从 Pi-hole 导出 jego 相关域名, `192.168.1.123` 是手机 IP
```bash
 sqlite3 pihole-FTL.db "select distinct
 domain from queries where client='192.168.1.123' and domain like '%jego%'"
```
得到几个域名
> app1.jegotrip.com.cn
tms.jegotrip.com.cn
jegotrip-stage.cn-hongkong.log.aliyuncs.com
cdn.jegotrip.com.cn
oss.jegotrip.com.cn
cp.jegotrip.com.cn

逐一排查后，__tms.jegotrip.com.cn__ 解析到 223.118.41.228, 是香港 IP。

## 更新 IP 列表
223.118.41.228 属于 AS58453 自治域，把这个自治域中的几个 IP 段全部追加到国内 IP 列表。
```
223.118.0.0/15
223.119.0.0/16
223.120.0.0/16
223.121.0.0/16
```
重启梯子服务后问题解决。
