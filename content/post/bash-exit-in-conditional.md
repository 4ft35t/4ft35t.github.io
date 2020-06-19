---
title: "bash 条件退出踩坑"
date: 2020-06-19T17:55:19+08:00
tags: ["bash", "exit", "test", "shell"]
categories: ["Linux"]
draft: fasle

---

一个小脚本，`[ $status != "success" ] && exit ２`,在 status 值是 success 时，最终的返回状态码是１，而不是０，有点诡异。

<!--more-->

### 排查

这个写法以前在脚本里经常写，稳定运行未出错。对比之前的脚本后发现差异，之前的脚本 exit 是提前退出，后面还有内容；这个脚本exit是上层调用脚本反馈最终结果，exit 是最后一行。

修改脚本后一切正常

```bash
status=success
[ $status != "success" ] && exit ２
echo
```



### 原因分析

使用　bash -x 可以看出差异，

exit　在最后一行

```bash
bash -x t.sh
+ status=success
+ '[' success '!=' success ']'
```

exit 不在最后一行

```bash
bash -x t.sh
+ status=success
+ '[' success '!=' success ']'
+ echo
```

shell 会将最后一条命令的返回状态码作为整个脚本的返回状态码。test 在最后一行时，`[ $status != "success" ]` 的返回结果非0，后面的　&& 内容不会执行，整个脚本结束，脚本返回状态码就是中括号内test命令的反馈状态码。　　　　　　

### 避坑经验

退出用__||__, 避免用　__&&__；一定要用__&&做退出条件时，不能作为脚本或者函数的最后一条命令__。

上面脚本改造成||退出

```bash
[ $status = "success" ] || exit ２
```

因为是最后一行，还可以直接省略后面的内容

```bash
[ $status = "success" ]
```

