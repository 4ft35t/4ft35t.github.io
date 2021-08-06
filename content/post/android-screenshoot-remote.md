---
title: "Android 远程截图"
date: 2021-08-06T16:51:31+08:00
tags: ["android", "screenshoot", "autojs", "cgi", "adb", "scrcpy"]
categories: ["Network", "Android"]
draft: false
---
某些日常的特殊场景，需要远程截屏并传输，记录下使用 autojs + cgi + adb 实现过程。
<!--more-->

## 背景
云闪付公交乘车码，每周使用三次可以获得一张 6-5 消费券，但那一个帐号一周只能领一次，需要多个帐号。

家里有个老华为手机，上面登录的有老人的云闪付帐号，但是注册帐号的手机号失效，无法更换新手机号，新设备登录需要短信验证码，只有拿这个手机去刷。

不想带多个手机出门，开启折腾之路。

## 截屏
系统自带 `screencap` 命令，桌面和别的 app 可以截屏，云闪付不行，应该是云闪付设置了`FLAG_SECURE`。

[scrcpy](https://github.com/Genymobile/scrcpy) 可以显示云闪付内容，猜测 [autojs](https://github.com/Ericwyn/Auto.js) 使用 cast 模式也可以截屏，实测确实可以。
```js
auto();
app.launch("com.unionpay");
waitForPackage("com.unionpay");

// 首页点击乘车码
sleep(1500)
if(text("充值中心").exists()){
    toastLog("找到 乘车码 关键字");
    target = text("乘车码").findOne();
    while(!target.clickable()){
        target = target.parent()
    }
    target.click()
}

waitForActivity("com.unionpay.activity.react.UPActivityReactNative");
sleep(2000);
// 2、请求截图
if(!requestScreenCapture()){
    toastLog("请求截图失败");
    exit();
}

// 3、进行截图
captureScreen("/sdcard/img.png");
toastLog("截图完成");
```

## 触发
一开始设想的是 autojs 监听通知，使用 [Bark](https://github.com/Finb/Bark) 的安卓客户端 [PushLite](https://github.com/xlvecle/PushLite) 来接收通知触发截图，然而 PushLite 需要 Google FCM，安装太折腾，放弃，寻求电脑端触发的方案。

一番搜索，决定用 cgi 来接收外部请求并触发截图，各种依赖最少。
电脑上连接多个安卓手机，执行 adb 命令需要指定设备 id，设备 id 用 `adb devices` 获取
```bash
adb devices
List of devices attached
NXTDU1xxxxxxxxx        device
```
NXTDU1xxxxxxxxx 就是设备 id

yunshanfu.sh
```bash
#!/bin/bash
echo "Content-Type:text/html"
echo ""
echo '<html><meta charset="UTF-8"><body>'

dev=NXTDU1xxxxxxxxx
img=/sdcard/img.png

adb(){
    /usr/bin/adb -s $dev $@
}

cleanup(){
    adb shell rm $img
    rm /tmp/ysf.png
}

unlock(){
    adb shell input keyevent 224
    adb shell input swipe 300 500 300 1500
    adb shell input text 12345678
}

screenshoot(){
     adb shell am start -n org.autojs.autojs/.external.open.RunIntentActivity -d /sdcard/Scripts/ysf.js
     sleep 3
     while true
     do
         # 校验文件大小，避免 pull 回来未写完图片
         adb pull $img /tmp/ysf.png && [ "$(stat -c %s /tmp/ysf.png)" -ge 300000 ] && break
         sleep 1
     done

     # press power button
     adb shell input keyevent 26
}

cleanup
unlock
screenshoot

echo "<img src='data:image/png;base64,$(cat /tmp/ysf.png | base64 -w 0)'>"
echo "</body></html>"
```

python http.server 的 cgi 脚本需要放在 `cgi-bin` 目录
```bash
chmod +x yunshanfu.sh
mkdir cgi-bin
mv yunshanfu.sh cgi-bin
python3 -m http.server --cgi
```
浏览器访问 `http://127.0.0.1:8000/cgi-bin/yunshanfu.sh`, 即可触发截图，并在当前页面展示截图。
设置端口转发，公网访问效果如图
![](https://cdn.jsdelivr.net/gh/4ft35t/images@blog/img/2021/IMG_0317.jpg)

最后，所有相关代码在 https://github.com/4ft35t/Android-Screenshoot-Remote
