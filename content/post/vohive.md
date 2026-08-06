---
title: "Vohive 与 WIFI Calling"
description: 
date: 2026-08-06T21:33:38+08:00
image: https://miro.medium.com/v2/resize:fit:180/format:webp/1*ihirvzvqsu62iOt44hx1lg.jpeg
math: 
hidden: false
comments: true
tags: [vohive, WIFI Calling, vowifi, sms]
categories: [Linux]
draft: false
---

vohive 是一个使用高通4G/5G芯片，实现WIFI Calling（VoWiFi）功能的工具, 主要用于短信转发，特别是使用 WIFI Calling 收短信。详细功能介绍见 https://www.nodeseek.com/post-610499-1。

vohive 不开源，只提供二进制文件，支持 Linux 系统。

作者已被喷子气跑，删库跑路了，https://github.com/iniwex5/vohive-release。

## 重要提示
**有 WIFI Calling 需求的用 vohive 合适，只有短信转发需求的用 https://github.com/chenxuuu/sms_forwarding 更便宜更稳定。**

## 支持的硬件
- 移远的 EC20 系列模块
- 高通 410 的随身 wifi
- 大疆一代 4g 模块

### EC20 系列模块
EC20 系列模块是 vohive 最早支持的硬件, 需要单独购买模块、转接板和天线，目前已不推荐。

手头有存货的可以折腾使用，购买不推荐。

使用电信网络需要数最新的 R08 版本固件，群内分享的固件已整理放在 https://github.com/4ft35t/4ft35t.github.io/releases/tag/ec20-firmware

### 高通 410 的随身 wifi
最便宜的方案，手头有存货的可以折腾使用，购买不推荐，毕竟做工在那里，寿命有限。

折腾过程：
1. MiKo 完整备份
2. 切换外置卡槽，具体按板子上的型号去搜索，都有资料
3. 使用 https://github.com/OpenStick/OpenStick 刷 debian，具体操作见项目 wiki
4. 最重要的一步，恢复 modem 固件。 从第一步的备份中找到 modem.bin 或者 modem.img, 然后使用 fastboot flash modem modem.bin 或者 fastboot flash modem modem.img 恢复。

### 大疆一代 4g 模块
最推荐的方案，折腾最少，只需要买模块，无需转接板和天线。

咸鱼搜“大疆一代4g模块”，当时35包邮，目前最便宜的 64。

### 刷机
大疆一代 4g 模块，实际是移远的 EC25 模块，
大疆模块用 usb 线插入 linux 电脑，拔掉电脑上其余 USB 设备, 执行以下命令一次，即可刷回 EC25 模块，在所有机器上免驱使用。
```bash
# 强行挂载串口
modprobe usbserial
echo "2ca3 4006" > /sys/bus/usb-serial/drivers/generic/new_id
# 切换模式
echo -e 'AT+QCFG="usbcfg",0x2C7C,0x0125,1,1,1,1,1,0,0\r' > /dev/ttyUSB2
# 重启模块
echo -e "AT+CFUN=1,1\r" > /dev/ttyUSB2
```
等几秒后执行 `lsusb`，看到 `2c7c:0125 Quectel` 就说明刷机成功了。

## vohive 使用
### 安装
1. `sudo mkdir -p /opt/vohive/{config,data,logs}`
2. 从 https://github.com/4ft35t/4ft35t.github.io/releases/tag/vohive 下载 vohive 二进制文件, 重命名为 vohive, 放到 /opt/vohive 目录下
3. `sudo chmod +x /opt/vohive/vohive`
4. `sudo echo -e 'server:\n  port: 7575\n  debug: false\n web:\n  username: admin\n  password: admin' > /opt/vohive/config/config.yaml`
5. 启动, 必须使用 root， 否则找不到 4g 模块。`sudo /opt/vohive/vohive -c /opt/vohive/config/config.yaml`

### 使用
在网页中访问 http://localhost:7575, 输入用户名和密码 admin/admin 登录，点击“添加设备”，选择对应的模块，输入 ID，内容随意，几个模块不重复即可，点击“保存”。

等几秒就能看到设备状态变为“在线”。

具体的短信转发通道在网页里都有，不再赘述。

### 开启 WIFI Calling
开启 WIFI Calling 需要运营商支持，同时需要有对应国家的梯子出口，梯子需要支持 UDP 。

不建议搞软路由的全局分流，推荐在运行 vohive 的机器上直接开梯子，然后在代理管理-漫游前置代理中添加梯子监听的 socks5 代理，保存后点“国家规则”，把代理绑定到对应的国家。

最后去设备管理-对应模块-配置-VoWiFi，开启 VoWiFi-保存，等待几秒，设备状态变为“在线”，就可以使用 WIFI Calling 了。

部分运营商会挑 IP 质量，如果出现无法注册的情况，可以尝试给梯子套 cloudflare warp。
