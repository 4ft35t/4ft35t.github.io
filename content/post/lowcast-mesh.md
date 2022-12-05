---
title: "低成本 mesh 组网"
description: 偶然在什么值得买看到“最便宜的组Aimesh的方案”，正好有网件6300V2，动手实践一番
date: 2022-12-04T17:57:03+08:00
image: 
math: 
hidden: false
comments: true
tags: [wifi, mesh, 6300v2, 梅林]
categories: [Network]
draft: false
---

上一次主动看 mesh 路由，单台成本 2000 左右，最近在什么值得买上看到用网件6300V2刷机后租mesh的，二手价格单台不到80，立马买两台来组网。

## 硬件成本
自有一台服役多年的 6300V2, 淘宝新买2台二手，77.99 一台包邮。

## 刷机
### 1. 从官方固件刷梅林
下载 380 版本梅林固件，https://fw.koolcenter.com/KoolCenter_Merlin_Legacy_380/Netgear/R6300V2/X7.9.1/R6300V2_380.70_0-X7.9.1-koolshare.chk

路由器恢复出厂设置:

路由器关机，按住reset开机，15秒后松开

浏览器打开 192.168.1.1，admin/password 登录，固件升级选择 R6300V2_380.70_0-X7.9.1-koolshare.chk

### 2. 备份 CFE 和 board_data
刷梅林 380 版本后，路由器地址变成 192.168.50.1。登录路由器, 帐号密码沿用 admin/password, 打开 SSH 服务

系统管理 - 系统设置 - 启用 SSH- LAN only - 应用本页面设置

使用 ssh 登录路由器，Windows 可以用 putty 或者 wsl。
```bash
dd if=/dev/mtd0 of=/tmp/backup_cfe.bin
dd if=/dev/mtd4 of=/tmp/backup_board_data.bin
```

使用 scp 把 上面两个文件拖回本地，如果需要就砖会用到。

如果遇到 `/usr/libexec/sftp-server: not found` 错误，是因为本地的 scp 版本太新，加 `-O` 参数即可

```
scp -O admin@192.168.50.1:/tmp/backup_cfe.bin .
```

### 3. 刷 CFE
下载 CEF https://fw.koolcenter.com/KoolCenter_Merlin_New_Gen_384/NETGEAR/R6300v2/tools/cfe_2.4Gfix.bin

下载 CEF 修改器 https://fw.koolcenter.com/KoolCenter_Merlin_New_Gen_384/NETGEAR/R6300v2/tools/CFEEdit.exe

打开 CFEEdit.exe, 把 MAC 地址修改成华硕的地址段

需要修改五个位置
```
et0macaddr=00:90:4C:0F:F1:A7
0:macaddr=00:90:4C:0F:F1:A7
1:macaddr=00:90:4C:0F:F1:AB
secret_code=73210917
odmpid=RT-AC68U
```
另存为 cfe-1.bin

继续修改，把上面内容 MAC 地址中倒数第二段的 `F1` 分别修改成 `F2` 和 `F3`，同时 `secret_code` 最后的 `17` 改成 `27` 和 `37`, 另存为 cfe-2.bin 和 cfe-3.bin， 分别对应三台机器。

scp 把 CFE 传送到路由器
```
scp -O cfe-1.bin admin@192.168.50.1:/tmp
```

ssh 登陆路由器，开始刷 CFE
```
dd if=/tmp/cfe-1.bin of=/dev/mtd0
nvram erase
reboot
```

重启后路由器会进入 tftp 刷机状态，ip 为 192.168.1.1

### 4. tftp 刷入梅林 386 版本
下载 386 版本固件 https://fw.koolcenter.com/KoolCenter_Merlin_New_Gen_386/Netgear/R6300v2/RT-AC6300v2_386.2_4-20210628-91625500b.trx

Windows 电脑安装自带的 tftp 客户端

Win+r - 输入 appwiz.cpl - 打开或关闭Windows功能 - 勾选TFTP客户端 - 重启

打开一个 cmd 窗口，执行 `ping -t 192.168.1.1` 观察 TTL，TTL 值为 100 即表示路由器处于 tftp 刷机状态，如果不是 100，重启路由器
打开第二个 cmd 窗口, 执行 `tftp -i 192.168.1.1 put RT-AC6300v2_386.2_4-20210628-91625500b.trx`

等 tftp 传输完成后，路由器会在3分钟内重启多次，最后完成刷机。

当 ping 的 TTL 变成 64， 即表示路由器正常启动。如果 tftp 传输完成后5分钟，TTL 还没变成 64，重启路由器。

如果重启后 ping TTL 值一直保持 100，浏览器访问 192.168.1.1， 进入 CFE miniWeb，点击 "Restore default NVRAM values", 然后重启。

### 4. 刷机后存在问题
刷机后，WAN 口的位置会发生变化
```
+------+------+------+------+   +------+
| LAN4 |LAN3  | LAN2 | LAN1 |   |  WAN |
|      |      |      |      |   |      |
+------+------+------+------+   +------+

+------+------+------+------+   +------+
| WAN  |LAN1  |LAN2  |LAN3  |   |LAN4  |
|      |      |      |      |   |      |
+------+------+------+------+   +------+
```

## mesh 组网
路由1 做为 mesh 路由，访问 192.168.1.1，按向导完成 pppoe 拨号设置, 选路由器模式。


路由器2，作为 AiMesh 节点，按向导完成动态动态 IP 设置；然后

系统管理 - 操作模式 - AiMesh 节点 - 保存

等界面显示重置完成，路由器2的设置完成。

现在电脑网线连接路由器1；路由器2保持开机，不插网线，放在路由器1旁边。

登录路由器1网页 - AiMesh - 添加 AiMesh 节点 - 选择路由器2（RT-AC86U） - 建立连接


等待完成后对路由器3重复执行上述操作。

所有操作完成后，路由器2和3的用户名密码会变成和路由器1一样。

现在可以享受 mesh 组网的无缝切换了，不用再像用扩展器一样手动切换。

### 有线回程
路由器1 LAN 连接路由器2 WAN，即可让路由器2 变成有线回程。路由器3 同理。


## 参考链接
- [最便宜的组Aimesh的方案——R6300V2最后的荣光](https://post.smzdm.com/p/a4dk3zgk/)
- [网件NETGEAR R6300V2(国行版本) 原厂固件直刷380_X7.9.1梅林成功 附升级梅林386教程 ](https://www.right.com.cn/forum/thread-7613019-1-1.html)
- [网件路由器 R6300v2 刷机教程和固件大全](https://wkings.blog/archives/875)
- [无线路由器Mesh组网教程：华硕Aimesh组网篇](https://zhuanlan.zhihu.com/p/386867842)

