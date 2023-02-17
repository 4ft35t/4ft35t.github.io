---
title: "Ubuntu 20.04 server iptables 持久化"
description: 
date: 2023-02-17T10:36:40+08:00
image: https://i.stack.imgur.com/gZm6i.png
math: 
hidden: false
comments: true
tags: [iptables, ipset, netfilter, persistent, debconf, noninteractive]
categories: [Network]
draft: false
---

有一批服务器需要做 iptables 持久化，开机自动加载。

在 Ubuntu 12.04 时代, 可以把加载脚本写到 `/etc/rc.local` 开机自动加载，Ubuntu 20.04 要启用 rc.local 太麻烦，用 `netfilter-persistent` 来实现自动加载。

`netfilter-persistent` 加载 `ipset-persistent` 和 `iptables-persistent` 保存的规则。

## TLDR
```bash
# noninteractive install
sudo debconf-set-selections <<EOF
iptables-persistent iptables-persistent/autosave_v4 boolean true
iptables-persistent iptables-persistent/autosave_v6 boolean true
ipset-persistent ipset-persistent/autosave boolean true
EOF

sudo apt update && sudo apt install netfilter-persistent ipset-persistent iptables-persistent  -y

sudo netfilter-persistent save
sudo systemctl enable netfilter-persistent
sudo systemctl start netfilter-persistent
```

## 静默安装
ipset-persistent 和 iptables-persistent 安装过程中会弹出交互窗，选择是否自动保存规则。

交互会打断服务器上脚本安装，需要无交互静默安装。


Ubuntu 上软件包安装配置项归 `debconf` 管理，在一台机器上交互安装完成后，用 `debconf-show` 查看软件包的配置项。

```bash
sudo debconf-show ipset-persistent
* ipset-persistent/autosave: true

sudo debconf-show iptables-persistent
* iptables-persistent/autosave_v4: true
* iptables-persistent/autosave_v6: true
```

在安装前，用 `debconf-set-selections` 在 debconf 数据库中插入对应的内容，即可实现无交互静默安装。

debconf-set-selections 配置语法
```
{包名} {配置项key} {配置项类型} {配置项value}
```

生成的配置如下
```bash
sudo debconf-set-selections <<EOF
ipset-persistent ipset-persistent/autosave boolean true
iptables-persistent iptables-persistent/autosave_v4 boolean true
iptables-persistent iptables-persistent/autosave_v6 boolean true
EOF
```
执行 ` sudo apt install netfilter-persistent ipset-persistent iptables-persistent  -y` 即可静默安装。

### 持久化
#### 保存规则
`sudo netfilter-persistent save`

#### 自动加载
```bash
sudo systemctl enable netfilter-persistent
sudo systemctl start netfilter-persistent
```

## 参考链接
- [Make ipset and iptables configurations persistent in Debian/Ubuntu](https://dhtar.com/make-ipset-and-iptables-configurations-persistent-in-debianubuntu.html)
- [Installing iptables-persistent on ubuntu without manual input](https://gist.github.com/alonisser/a2c19f5362c2091ac1e7)
- [Linux下实现软件的静默安装](https://blog.51cto.com/u_13791715/2308514)
- [How to read and insert new values into the debconf database](https://sleeplessbeastie.eu/2018/09/17/how-to-read-and-insert-new-values-into-the-debconf-database/)
