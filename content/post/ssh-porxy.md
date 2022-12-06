---
title: "SSH 跳板机与代理"
image: https://www.openssh.com/images/openssh.gif
date: 2020-06-03T16:55:35+08:00
tags: [ssh, proxy, netcat, socat, connect, corkscrew]
categories: [Network]
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

  `ProxyCommand ssh jumpserver nc %h %p`

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

- nc `ProxyCommand nc -X connect -x 1.2.3.4:3128 %h %p`

- https://github.com/bryanpkc/corkscrew `ProxyCommand /usr/local/bin/corkscrew 1.2.3.4 3128 %h %p`

__corkscrew 在代理不稳定时比 nc 可靠__ 

### 特别注意

本文提到的 nc 是 **BSD 版本的 netcat**， GNU 版本的 natcat 没有代理功能。

更多 netcat 版本见 https://wiki.archlinux.org/title/Network_tools#Netcat

### 使用 socat 突破某些堡垒机
某堡垒机的OpenSSH版本是`OpenSSH_7.8p1`, 使用`ssh -J jumpserver user@dst-host`无法连接，从报错信息来看，堡垒机禁止了 SSH 转发功能
```bash
debug1: Local version string SSH-2.0-OpenSSH_8.1
Stdio forwarding request failed: Session open refused by peer
kex_exchange_identification: Connection closed by remote host
```
`ssh -tt` 和 `ssh -W` 也是同样错误，SSH 自身转发走不通，需要借助第三方软件。

#### 尝试1 - 使用 nc 转发
使用 `ProxyCommand ssh jumpserver nc %h %p` 连接失败，报错, 跳板机上没有 nc
```bash
debug1: Local version string SSH-2.0-OpenSSH_8.1
bash: nc: command not found
```
用包管理器安装 netcat, 提供的只有 nmap 版本，没有代理功能。

#### 尝试2 - 使用 socat 转发
Socat 是 Linux 下的一个多功能的网络工具，名字来由是 「Socket CAT」。其功能与有瑞士军刀之称的 Netcat 类似，可以看做是 Netcat 的加强版。Socat 的主要特点就是在两个数据流之间建立通道，支持众多协议和链接方式，如 IP、TCP、 UDP、IPv6、PIPE、EXEC、System、Open、Proxy、Openssl、Socket等。


使用 `ProxyCommand ssh jumpserver socat - tcp:%h:%p` 成功通过堡垒机连接目标机器。

### socat 在ProxyCommand 中与 netcat 等效用法
socat 稳定版 1.7，beta 版 2 的代理功能更丰富。

#### 跳板机/堡垒机
`ProxyCommand ssh jumpserver nc %h %p` -> `ProxyCommand ssh jumpserver socat - tcp:%h:%p`

#### socks 代理
`ProxyCommand nc -x 127.0.0.1:1080 %h %p` ->
- socat1 `ProxyCommand socat - "SOCKS4:127.0.0.1:%h:%p,socksport=1080"`
- socat2 `ProxyCommand socat - "SOCKS5:%h:%p|tcp:127.0.0.1:1080"`

#### HTTP/HTTPS 代理
`ProxyCommand nc -X connect -x 1.2.3.4:3128 %h %p` ->
- socat1 `ProxyCommand socat - "PROXY:1.2.3.4:%h:%p,proxyport=3128"`
- socat2 `ProxyCommand socat - "PROXY:%h:%p|tcp:1.2.3.4:3128"`

### 参考链接
- https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts
- https://www.cyberciti.biz/faq/linux-unix-ssh-proxycommand-passing-through-one-host-gateway-server/
- https://www.endpointdev.com/blog/2013/04/socat-and-netcat-proxycommand-ssh/
- https://unix.stackexchange.com/questions/68826/connecting-to-host-by-ssh-client-in-linux-by-proxy
- https://gitolite.com/git-over-proxy.html
- https://linux.die.net/man/1/socat
