---
title: "Promark3 使用 冰人（iceman）固件"
image: http://img-blog.csdnimg.cn/20200307161927634.jpg
date: 2020-06-04T19:21:59+08:00
tags: ["rfid", "error", "proxmark3"]
categories: ["Mess"]
draft: false
---



淘宝买的 Promark3 RDV2，使用官方仓库的代码https://github.com/Proxmark/proxmark3，编译和刷固件都没问题；使用冰人的固件https://github.com/RfidResearchGroup/proxmark3，编译后刷bootrom.elf 没问题，刷 fullimage.elf 出错 `Error: Unexpected reply 0x00fe NACK (expected ACK) Lock Error`。

<!--more-->

官方固件编译、烧录、使用都没问题，看到冰人的固件有两条命令很心动，把重复工作自动化，节约时间。

- `auto` 自动识别卡类型
- `hf mf autopwn`  自动破解 IC 卡密码



## 编译安装

win 10 的 WSL 可以直接访问串口，很方便。win 10 上的 COM3， 自动映射到 WSL 的 /dev/ttyS3。

```bash
sudo apt-get install --no-install-recommends git ca-certificates build-essential pkg-config libreadline-dev gcc-arm-none-eabi libnewlib-dev

git clone https://github.com/RfidResearchGroup/proxmark3.git
cd proxmark3
make clean && make all
```

修改 /dev/ttyS3 为所有用户可读写，设备拔插后需要再次执行

`sudo chmod 666 /dev/ttyS3`

同时刷 bootimg 和 fullimage

`./pm3-flash-all`

bootimg 刷成功，刷 fullimage 出错

> Error: Unexpected reply 0x00fe NACK (expected ACK)
>        Lock Error

![image-20200604192556929](https://cdn.jsdelivr.net/gh/4ft35t/images@blog/img/2020/20200604192557.png)



## 刷预编译固件

在冰人固件的 wiki 中找到 windows 的预编译非 RDV4 版本下载，Generice Proxmark3 devices (non RDV4) [[Precompiled builds for RRG / Iceman repository x64](https://drive.google.com/open?id=1PI3Xr1mussPBPnYGu4ZjWzGPARK4N7JR)

提取 fullimage.elf 后，顺利刷入。

```bash
kali@DESKTOP-SF5P:~/iceman/proxmark3$ ./pm3-flash  ./fullimage.elf
[=] Session log /home/kali/.proxmark3/log_20200604.txt
[=] Loading Preferences...
[+] loaded from JSON file /home/kali/.proxmark3/preferences.json
[+] About to use the following file:
[+]    ./fullimage.elf
[+] Waiting for Proxmark3 to appear on /dev/ttyS5
 🕑  59 found
[=] Available memory on this board: 512K bytes

[=] Permitted flash range: 0x00102000-0x00180000
[+] Loading ELF file ./fullimage.elf
[+] Loading usable ELF segments:
[+]    0: V 0x00102000 P 0x00102000 (0x0003b4d0->0x0003b4d0) [R X] @0x94
[+]    1: V 0x00200000 P 0x0013d4d0 (0x00001370->0x00001370) [RW ] @0x3b564
[=] Note: Extending previous segment from 0x3b4d0 to 0x3c840 bytes

[+] Flashing...
[+] Writing segments for file: ./fullimage.elf
[+]  0x00102000..0x0013e83f [0x3c840 / 485 blocks]
...................................................................
        @@@  @@@@@@@ @@@@@@@@ @@@@@@@@@@   @@@@@@  @@@  @@@
        @@! !@@      @@!      @@! @@! @@! @@!  @@@ @@!@!@@@
        !!@ !@!      @!!!:!   @!! !!@ @!@ @!@!@!@! @!@@!!@!
        !!: :!!      !!:      !!:     !!: !!:  !!! !!:  !!!
        :    :: :: : : :: :::  :      :    :   : : ::    :
        .    .. .. . . .. ...  .      .    .   . . ..    .
................................................ OK

[+] All done

Have a nice day!
```

## 解决方案

到这一步可以确认是关键编译参数问题，而非硬件缩水问题233。

查阅文档后，默认编译的目标平台是 PM3RDV4，非 PM3RDV4 需要设置为 PM3OTHER。

```bash
echo 'PLATFORM=PM3OTHER' > Makefile.platform
make clean && make all
```

再刷固件就一切顺利

```bash
kali@DESKTOP-SF5P:~/iceman/proxmark3$ ./pm3-flash-fullimage
[=] Session log /home/kali/.proxmark3/logs/log_20200604.txt
[=] Loading preferences...
[+] loaded from JSON file /home/kali/.proxmark3/preferences.json
[+] About to use the following file:
[+]    /home/kali/iceman/proxmark3/client/../armsrc/obj/fullimage.elf
[+] Waiting for Proxmark3 to appear on /dev/ttyS5
 🕑  59 found
[+] Entering bootloader...
[+] (Press and release the button only to abort)
[+] Waiting for Proxmark3 to appear on /dev/ttyS5
 🕒  48 found
[=] Available memory on this board: 512K bytes

[=] Permitted flash range: 0x00102000-0x00180000
[+] Loading ELF file /home/kali/iceman/proxmark3/client/../armsrc/obj/fullimage.elf
[+] Loading usable ELF segments:
[+]    0: V 0x00102000 P 0x00102000 (0x0003b240->0x0003b240) [R X] @0x94
[+]    1: V 0x00200000 P 0x0013d240 (0x00001360->0x00001360) [RW ] @0x3b2d4
[=] Note: Extending previous segment from 0x3b240 to 0x3c5a0 bytes

[+] Flashing...
[+] Writing segments for file: /home/kali/iceman/proxmark3/client/../armsrc/obj/fullimage.elf
[+]  0x00102000..0x0013e59f [0x3c5a0 / 483 blocks]
...................................................................
        @@@  @@@@@@@ @@@@@@@@ @@@@@@@@@@   @@@@@@  @@@  @@@
        @@! !@@      @@!      @@! @@! @@! @@!  @@@ @@!@!@@@
        !!@ !@!      @!!!:!   @!! !!@ @!@ @!@!@!@! @!@@!!@!
        !!: :!!      !!:      !!:     !!: !!:  !!! !!:  !!!
        :    :: :: : : :: :::  :      :    :   : : ::    :
        .    .. .. . . .. ...  .      .    .   . . ..    .
.............................................. OK

[+] All done

Have a nice day!
```

刷完固件后的版本信息

>  [ ARM ]
>   bootrom: RRG/Iceman/master/v4.9237-188-gbd5aa92a 2020-06-04 16:05:12
>        os: RRG/Iceman/master/v4.9237-188-gbd5aa92a 2020-06-04 21:01:01
>   compiled with GCC 8.3.1 20190703 (release) [gcc-8-branch revision 273027]

## 参考链接

- https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/md/Installation_Instructions/Windows-Installation-Instructions.md#compile-and-use-the-project-1

- https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/md/Use_of_Proxmark/4_Advanced-compilation-parameters.md
