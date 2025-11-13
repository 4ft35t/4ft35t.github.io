---
title: "群晖 DSM7 上使用古董打印机 HP CP1025"
description: 
date: 2025-10-23T17:29:21+08:00
image: https://ssl-product-images.www8-hp.com/digmedialib/prodimg/lowres/c02768865.png
math: 
hidden: false
comments: true
tags: [hp, cp1025, airprint, cups, docker, dsm, synology]
categories: [Linux]
draft: false
---

想在群晖 DSM7.2 上使用 老古董 HP CP1025 打印机, 但群晖自带的打印服务器不支持该打印机驱动, 只能通过 Docker 安装 CUPS 来实现.

<!--more-->

搜索多篇文章，找到几个 docker 镜像，大部分无法识别 HP CP1025 打印机, 只有 chuckcharlie/cups-avahi-airprint 能找到打印机。


文章中提到的几个 docker 镜像:
 - olbat/cupsd 找不到打印机
 - ydkn/cups 找不到打印机
 - chuckcharlie/cups-avahi-airprint 能找到打印机


chuckcharlie/cups-avahi-airprint 能找到打印机, 但无法打印，提示 "Filter failed"。

一番搜索后，在[树莓派论坛](https://forums.raspberrypi.com/viewtopic.php?t=118732)找到了原因， hplip 驱动有问题，需要安装 foo2zjs 开源驱动。


### 玩客云测试
为了测试 foo2zjs 驱动是否有效， 打开了另一件吃灰的老古董玩客云, 系统是 armbian, 安装了 openmediavault 4。

从 omv-extras 里安装 openmediavault-cups, 然后在 cups web 界面添加打印机, 选择 "HP LaserJet Pro CP1025nw Foomatic/foo2zjs-z3 (en)" 驱动，成功打印测试页。

如果选择 "HP LaserJet Cp1025, hpcups 3.23.12 (en)" 驱动，打印测试页失败，提示 "Filter failed"。

确认 foo2zjs 驱动可以正常使用。


### 在群晖 DSM7 上安装 CUPS
#### 关闭自带 cups 服务
在群晖 DSM7 上, 需要先关闭自带的打印服务器服务, 否则会占用 631 端口.
```
sudo synosystemctl stop cupsd
sudo synosystemctl disable cupsd
systemctl mask cupsd
sudo synosystemctl stop cups-service-handler
sudo synosystemctl disable cups-service-handler
```

#### 运行 docker 容器
创建配置目录 ` mkdir -p /volume1/docker/airprint/config /volume1/docker/airprint/services `

创建 `/volume1/docker/airprint/docker-compose.yml` 文件:
```yaml
version: '3'
services:
  airprint:
    image: 'chuckcharlie/cups-avahi-airprint'
    privileged: true
    restart: unless-stopped
    environment:
      - TZ=Asia/Shanghai
      - CUPSADMIN=admin
      - CUPSPASSWORD=pass
    devices:
      - /dev/bus
      - /dev/usb
    volumes:
      - /var/run/dbus:/var/run/dbus
      - ./config/:/config
      - ./services/:/services
    ports:
      - '631:631'
```

启动容器: ` cd /volume1/docker/airprint && sudo docker-compose up -d `

#### 安装 foo2zjs 驱动
foo2zjs 官网 http://foo2zjs.rkkda.com 已经无法访问，幸好 webarchive.org 上有存档可以下载。
进入容器: ` cd /volume1/docker/airprint && sudo docker-compose exec airprint sh`
在容器内下载并安装 foo2zjs 驱动。

```
cd /tmp
wget https://web.archive.org/web/20210224094943if_/http://foo2zjs.rkkda.com/foo2zjs.tar.gz
tar zxf foo2zjs.tar.gz
cd foo2zjs
make
make install
```

安装完成后，运行 `make cups` 进行配置修改, 会修改 `/etc/cups/cups-files.conf` 文件, 允许使用 file 设备。
但是这里会依赖 `vim`，镜像里没有；对比修改前后的文件，发现只需要添加两行配置, 直接用 `echo` 命令添加即可。

`echo -e 'FileDevice Yes\nSandboxing Relaxed' >> /etc/cups/cups-files.conf`

修改完成后，重启容器。

### 添加打印机
打开浏览器访问 `http://群晖IP:631`, 进入 CUPS web 界面, 添加打印机。

Administration - Add Printer - 选择 HP CP1025 打印机 - 下一步勾选 "Share This Printer" - 选择驱动 "HP LaserJet Pro CP1025nw Foomatic/foo2zjs-z3 (en)" - 完成添加。

下一步修改打印机默认选项:
 - Color Mode: Color
 - Resolution: 600x600dpi
 - Paper Size: A4
 - Media Source: Auto Source
 - Media Type: Standard Paper

点击 "Set Default Options" 保存。

打印测试页，成功打印。

### 设置自动启动
打印从休眠中恢复，需要重启 docker 容器才能使用。
在群晖里创建`udev` 规则，检测到打印机连接时，自动重启 docker 容器。
#### 获取打印机属性
ssh 到群晖，运行 `sudo udevadm monitor --kernel --property --subsystem-match=usb`，然后插入打印机 USB 线，记录打印机的属性:

```
KERNEL[69024.879332] add      /devices/pci0000:00/0000:00:13.1/0000:02:00.0/usb3/3-1/3-1:1.0 (usb)
ACTION=add
DEVNAME=/dev/3-1:1.0
DEVPATH=/devices/pci0000:00/0000:00:13.1/0000:02:00.0/usb3/3-1/3-1:1.0
DEVTYPE=usb_interface
INTERFACE=7/1/2
MODALIAS=usb:v03F0p112Ad0001dc00dsc00dp00ic07isc01ip02in00
PHYSDEVBUS=usb
PRODUCT=3f0/112a/1
SEQNUM=2984
SUBSYSTEM=usb
TYPE=0/0/0
```
记录 `PRODUCT=3f0/112a/1`。

#### 创建 udev 规则
创建文件 `/lib/udev/rules.d/50-hp-cp1025.rules`，内容如下:
```
ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ENV{PRODUCT}=="3f0/112a/1", RUN+="/volume1/docker/airprint/restart-cups.sh"
```
上面的规则在检测到 HP CP1025 打印机连接时，运行 `/volume1/docker/airprint/restart-cups.sh` 脚本。

`/volume1/docker/airprint/restart-cups.sh` 脚本内容如下:

```bash
#!/bin/sh
cd /volume1/docker/airprint
docker-compose restart
```
这里已经完成了全部配置，局域网内的 Mac 和 iOS 设备都可以通过 AirPrint 使用 HP CP1025 打印机了，windows 设备也可以通过添加网络打印机使用。


### 无线打印使用
### iOS 设备
iOS 设备可以直接通过 AirPrint 使用打印机，连接 wifi 后自动搜索网络打印机，无需额外配置。需要打印的文件，点击分享按钮，选择打印机即可。
### Mac 设备
同样无需设置，自动出现在打印机列表中，直接选择打印机打印即可。
### Windows 设备
Windows 设备需要去惠普网站下载安装 HP CP1025 驱动，然后添加网络打印机。
控制面版 - 设备和打印机 - 添加打印机 - 选择 "我需要的打印机不在列表中" - 选择 "使用 TCP/IP 地址或主机名添加打印机" - 设备类型选择 "TCP/IP 设备" - 主机名或 IP 地址填写群晖 IP 地址


### 参考链接
- [记一次群辉dsm6.2.3使用cups实现老款打印机cp1025](https://www.bilibili.com/opus/774732308492058647)
- [群晖、威联通NAS实现共享打印机+Airprint隔空打印教程，Docker版CUPS，让NAS变身打印服务器！](https://post.smzdm.com/p/ag4p7k7m/)
- [通过Docker的cups实现定期打印，防止爱普生墨仓式打印机堵墨](https://post.smzdm.com/p/a5xwr0x7/)
- [群晖dsm7.1 实现老款打印机AirPrint](https://www.bilibili.com/opus/720655857020305463)
- [CP1025 printer](https://forums.raspberrypi.com/viewtopic.php?t=118732)
- [DOCKER安装CUPS容器实现无线打印（支持大部分ARM设备](https://blog.zhoujie218.top/archives/2540.html)

