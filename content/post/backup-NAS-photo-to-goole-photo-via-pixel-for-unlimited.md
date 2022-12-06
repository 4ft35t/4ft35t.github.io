---
title: "利用 Pixel 一代的无限存储备份 NAS 照片"
image: https://oss-cn-hangzhou.aliyuncs.com/codingsky/cdn/img/2019-07-25/dc29fbcc14b8c4acce61ba33a7f2af63.jpg
date: 2022-02-27T23:41:49+08:00
tags: ["bash", "NAS", "photo", "shell", "google", "android", "autojs"]
categories: ["Linux"]
draft: false
toc: true

---

Pixel 一代可以无限上传原始质量的照片和视频到 Google Photos，是 NAS 上照片异地备份的好选择。
<!--more-->

## TLDR
### 手机端设置
1. Pixel 手机无需 root，安装 [Solid Explorer](https://play.google.com/store/apps/details?id=pl.solidexplorer2), 开启 FTP 服务。FTP 默认端口 9999, 默认目录 /sdcard/Download/。
2. 在 FTP 目录放一张图片（使用 Solid Explorer 复制、浏览器从网上下载、FTP 上传均可），Google Photos 会询问是否备份此目录，选备份。

### NAS 设置
群晖自带的 curl 不支持 FTP，需要下载一个静态编译开启 FTP 支持的 curl。可以在 [https://github.com/moparisthebest/static-curl/releases](https://github.com/moparisthebest/static-curl/releases) 下载。
1. 下载 curl 并重命名成 __curl-amd64__, 并执行 `chmod +x curl-amd64`
2. 下载上传脚本[https://gist.github.com/4ft35t/8024c8815a115ec134dd15965ed47fc5](https://gist.github.com/4ft35t/8024c8815a115ec134dd15965ed47fc5), 和 curl-amd64 一起放到家目录。
3. 修改 photo-upload.sh 中配置
  - ftp_server 手机 FTP server 地址，Solid Explorer 中可以看到
  - limit_size 脚本运行一次上传的图片/视频数量，过大会导致手机存储空间被占满
  - PHOTO_DIRS 19 行不动，修改 16 行，一行一个目录。我的群晖照片总库 `/volume1/photo`, 同时各用户使用 Synology Photos 备份手机照片到自己 home 目录, 17 行会加载所有用户目录下的照片，无需单独配置。
  - 控制面板-任务计划-新增-计划的任务-用户定义的脚本
    - 用户帐号选 `admin`
    - 计划-运行频率-每小时
    - 任务设置-用户定义的脚本，`bash /volume1/homes/xx/photo-upload.sh`，xx 是自己在 NAS 上用户

点击确定后，右键-运行，即可在手机上看到上传的照片/视频。

脚本根据 NAS 上照片的修改时间来增量备份，需要定期去手机上释放空间，避免手机空间占满。具体操作路径：Google 相册-右上角头像-释放空间。

##  2022.03.20 更新，使用 Autojs 自动释放手机空间
### 环境准备
 - https://github.com/Ericwyn/Auto.js/releases/download/V4.1.1.Alpha2/autoJs-V4.1.1.Alpha2-common-armeabi-v7a-debug.apk
 - https://raw.githubusercontent.com/4ft35t/utils/master/autojs/utils.js
 - https://gist.github.com/4ft35t/8024c8815a115ec134dd15965ed47fc5#file-google-photo-free-up-space-js

### 操作步骤
 1. 安装 autojs, 运行 app， 并按照提示打开无障碍服务。
 2. 将 utils.js 和 google-photo-free-up-space.js 放到手机 /sdcard/Scripts 目录
 3. 打开 autojs，下拉刷新，点击 google-photo-free-up-space.js 右边的三角符号运行
 4. 确认正常后，点击 google-photo-free-up-space.js 右边三点--更多--定时任务，设置每天运行时间

 google-photo-free-up-space.js 中 `text()` 函数是在界面查找文本，如果系统语言不是英文，需要自行调整 21-22 行内容
 ```js {hl_lines=["21-22"]}
 auto();
 console.show()

 var utils = require('utils.js');

 pkgName='com.google.android.apps.photos'

 utils.startApp(pkgName)
 toastLog('启动' + pkgName + '，等待检测中')
 waitForPackage(pkgName)
 toastLog('检测到' + pkgName + '等待 Activity')

 waitForActivity("com.google.android.apps.photos.home.HomeActivity")
 toastLog('找到首页 Activity')
 // 点击头像
 id("og_apd_internal_image_view").waitFor()
 utils.click(id("og_apd_internal_image_view"))
 toastLog('找到头像，点击')

 // 点击 free up
 text("Free up space").waitFor()
 utils.click(text("Free up space"))
 toastLog('进入释放界面')
 sleep(500)

 // 点击释放按钮
 if (id("free_up_button").exists()) {
     utils.click(id("free_up_button"))
     }
toastLog('任务完成')

// 返回 app 主界面
back()

// 2 次返回退出 app
back(); back();

console.hide()
```

## 踩坑过程
最初的设想，termux 安装 rclone, 然后将 NAS 共享目录挂载到本地目录，最后在相册里备份本地目录。

~~__rclone 挂载 SFTP__~~

群晖开启 STFP 支持，手机上 termux 里配置 rclone 一切正常，到执行 rclone mount 时报 `/etc/fstab` 问题，具体记不清，因该是和手机没有 root 有关。


~~__rclone 挂载 WebDAV__~~

WebDAV 在 rclone 里不支持 rclone mount。

~~__挂载 SMB 共享文件夹__~~

找了几个文件管理器，都只能挂载给自己使用，不能提供给别的 app。只有 root 后才能全局挂载。

## 参考链接
1. [Choose the upload size of your photos & videos](https://support.google.com/photos/answer/6220791)
2. [How to use OG Pixel as an unlimited Google Photos uploader](https://www.reddit.com/r/GooglePixel/comments/l9m6nk/how_to_use_og_pixel_as_an_unlimited_google_photos/)
3. [Original Pixel Phone – The Unlimited Google Photos Uploader](https://repaynt.com/2021/02/original-pixel-unlimited-google-photos-uploader/)
4. [rclone SFTP configuration](https://rclone.org/sftp/)
5. [mount-cifs-in-android](https://pmiku.com/note/mount-cifs-in-android.html)
