---
title: "SSH 跳板机与代理"
date: 2020-06-03T16:55:35+08:00
tags: ["ssh", "proxy"]
categories: ["Network"]
draft: false
---

 基于各种复杂环境，ssh   无法直接连接目标主机，需要借助中间的跳板机或者代理来连接。通常使用 ssh-agent 来通过跳板机连接目标主机，使用 ProxyCommand 或者 ProxyJump 连接更方便。

<!--more-->

### 跳板机

在　`~/.ssh/config` 中给跳板机一个方便记忆的别名，比如　jumpserver

```config
Host jumpserver
    User root
    HostName 1.2.3.4
    Port 2222
    IdentityFile ~/.ssh/jump.pem
```

之后就可以通过　`ssh jumpserver` 登录到跳板机。

通过跳板机连接目标主机：

`ssh -o ProxyCommand="ssh -W %h:%p jumpserver" usr@10.0.0.1`

或者写入 config 文件，使用 `ssh dst` 直接登录目标主机

```config
Host dst
    User usr
    HostName 10.0.0.1
    Port 22
    ProxyCommand ssh -W %h:%p jumpserver
```



较新版本的 ssh 客户端可以使用 ProxyJump

`ssh -J jumpserver usr@10.0.0.1`

或者写入 config 文件

```config
Host dst
    User usr
    HostName 10.0.0.1
    Port 22
    ProxyJump jumpserver
```

不同版本 ssh 客户端的差异如下

__ProxyCommand 与 ProxyJump__
所有命令向后兼容，新版本可以使用老版本命令
- 远古 SSH 客户端，OpenSSH < 5.4

  `ProxyCommand ssh jumpserver exec nc %h %p`

  或者
  `ssh -tt usr1@Jumphost ssh -tt usr2@FooServer`

- 中古 SSH 客户端，5.4 <= OpenSSH <= 7.2

  `ProxyCommand ssh jumpserver -W %h:%p`

- 现代 SSH 客户端，OpenSSH >= 7.3

  `ProxyJump jumpserver`

  命令行使用
  `ssh -J jumpserver user@dst-host`



### 代理

#### socks 代理

- nc `ProxyCommand nc -x 127.0.0.1:1080 %h %p`

- https://github.com/larryhou/connect-proxy `ProxyCommand connect-proxy -S 127.0.0.1:1080 %h %p`

#### HTTP/HTTPS 代理

- nc `ProxyCommand nc -X connect -x proxyhost:proxyport %h %p`

- https://github.com/bryanpkc/corkscrew `ProxyCommand /usr/local/bin/corkscrew proxyhost proxyport %h %p`

__corkscrew 在代理不稳定时比 nc 可靠__ 

### 特别注意

本文提到的 nc 是 **BSD 版本的 netcat**， GNU 版本的 natcat 没有代理功能。

更多 netcat 版本见 https://wiki.archlinux.org/title/Network_tools#Netcat



### 参考链接

- https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts
- https://www.cyberciti.biz/faq/linux-unix-ssh-proxycommand-passing-through-one-host-gateway-server/
