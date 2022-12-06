---
title: "解决应用商店 0x80131500 错误"
image: https://img-blog.csdnimg.cn/20190629132640532.png
date: 2020-06-03T11:48:10+08:00
tags: ["0x80131500", "0x80072EFD", "error", "windows"]
categories: ["Windows"]
draft: fasle
---

Win 10 应用商店打开就报 0x80131500 错误，尝试各种方法都不行，最终使用 fiddler 曲线解决。
<!--more-->

尝试过的方法来自 https://answers.microsoft.com/zh-hans/windows/forum/all/%E4%BB%A3%E7%A0%81/cbbe7aaf-8f66-4779-89c8-3c74f5341c7b

- ~~按 “Windows 徽标键+R”，在运行窗口中，键入“WSReset.exe” 并点击 “运行”~~
- ~~设置→应用→找到microsoft store →高级→重置此应用~~
- ~~打开 IE 浏览器，点击设置，打开 Internet 选项，点击高级，并勾选 “使用SSL 3.0”、”使用 TLS 1.0“、”使用 TLS 1.1“、”使用 TLS 1.2“，应用后重启电脑~~
- ~~卸载并重装应用商店~~

最终靠知乎的这条答案曲线解决。

https://www.zhihu.com/question/310873087/answer/1127300614

### 操作步骤

1. 下载安装 fiddler https://telerik-fiddler.s3.amazonaws.com/fiddler/FiddlerSetup.exe
2. 打开fiddler---点击左上角winconfig---给 Microsoft Store 打勾---点save changes

![image-20200603120155534](https://cdn.jsdelivr.net/gh/4ft35t/images@blog/img/2020/20200603120719.png)



再次打开应用商店就能正常使用。唯一的缺陷是需要打开 fiddler 才能使用应用商店，不然会报 0x80072EFD 错误。
