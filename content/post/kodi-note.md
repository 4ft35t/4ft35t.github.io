---
title: "Kodi 设置调整笔记"
description: 记录一些藏的比较深的 Kodi 设置
date: 2022-09-18T11:13:39+08:00
image: https://kodi.tv/images/kodi-logo-with-text.svg
math: 
hidden: false
comments: true
tags: [kodi, hdmi]
categories: [Mess]
draft: false
---

Kodi是由XBMC基金会开发的开源媒体播放器，原名XBMC，Kodi可以运行在多种操作系统和硬件平台。Kodi支持硬件解码，可以在硬件的很便宜的、低功耗的系统上播放高清视频


## 字幕移动到视频外黑框
电视分辨率一般16:9, 比如1920x1080, 遇到比例不一致的视频，会填充黑框。

Kodi 默认字幕显示在视频内底部， 播放1918x802美剧时，视频上下有黑边，字幕在视频内底部。把字幕位置移动到黑框内，可以获得更好体验。

### 设置方法
设置-播放器-语言-字幕在屏幕上位置，从`固定`切换到`视频外下方`


## CoreELEC 自动唤醒电视问题
CoreELEC 19.x 在斐讯 N1 上存在电视关机后自动唤醒问题，需要调整 HDMI CEC 相关设置

### 设置方法
设置-系统-输入-外设-cec adapter-电视关闭时, 从`待机`切换到`暂停播放`

## 参考链接
- [请问有谁知道哪个版本的kodi能设置字幕的显示位置？（已自解决，方法在7楼）](https://www.hao4k.cn/thread-24008-1-1.html)
- [Settings/Player/Language](https://kodi.wiki/view/Settings/Player/Language#Subtitles)
- [[BUG] 19.x 会自动唤醒电视 #84](https://github.com/RuralHunter/CoreELEC/issues/84)
