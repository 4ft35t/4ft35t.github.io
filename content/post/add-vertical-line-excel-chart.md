---
title: "excel 折线图添加竖线"
description: 网上搜到的答案都是针对 Windows 版的操作，MAC 版 Office 功能有阉割，不适用。
date: 2023-03-15T13:44:24+08:00
image: https://cdn.jsdelivr.net/gh/4ft35t/images@blog/2023/03/upgit_20230315_1678862174.jpg
math: 
hidden: false
comments: true
tags: [office, excel, '竖线', '垂直线', '分割线']
categories: [Office]
draft: false
---
## 背景
一般用折线图来表示发展趋势，需要在折线图的某个位置添加竖线，用来强调在这里发生了重要的区分性事件。需要的效果如下图。

![chart-vertical-line-ok](https://cdn.jsdelivr.net/gh/4ft35t/images@blog/2023/03/upgit_20230315_1678862174.jpg)

## 实现方法
本文是在 MAC 版 Microsoft Office 上的操作，Windows 版操作见下方的参考链接。

### 带直线的散点图
如果已有折线图

折线图上右键 - 更改图标类型 - XY（散点图）- 带直线的散点图

如果当前没有折线图

菜单栏，插入 - XY（散点图）- 带直线的散点图

### 新增分割线数据
画一条分割线需要占用4个单元格。在表格的任意空白位置，用左右两列来填写。

用上图的2月14号举例，需要在2月14号位置画一条竖线
- 左侧两格都是需要划线的 X 值
- 右侧两格分别是0和折线图的最大值，表示竖线的高度范围，无先后顺序

| X | Y |
| --- | --- |
| 2023/2/14 | 0 |
| 2023/2/14 | 70 |

### 画分割线
散点图上右键 - 选择数据 - 点“+”
- X 值 选 X 列
- Y 值 选 Y 列

大功告成。

### 画多条分割线
画多条分割线，直接在 X和Y列填写内容即可。但是要注意，__不同分割线的数据之间要空一行，不然分割线会连起来__，变成下图的样子。

![chart-vertical-line-bad](https://cdn.jsdelivr.net/gh/4ft35t/images@blog/2023/03/upgit_20230315_1678862263.jpg)

正确的数据格式
| X | Y |
| --- | --- |
| 2023/2/14 | 0 |
| 2023/2/14 | 70 |
| &nbsp; | &nbsp; |
| 2023/2/17 | 0 |
| 2023/2/17 | 70 |

### 分割线上加文字
任意空白单元格写好文字 - 复制 - 任意空白单元格 - 右键 - 选择性粘贴- 粘贴图片链接 - 把图片拖到分割线合适位置。

修改原始单元格的内容、改文字颜色等，图片上内容也会随之变动。

## 参考链接
- [Excel折线图美化技巧 在折线图中添加贯通竖线分隔季度数据](https://www.bilibili.com/video/BV1ex411J7Q1)
- [Excel绘制可以随日期移动垂直于X轴的直线，竖线](https://www.bilibili.com/video/BV1p44y1j7oq)
- [How to add vertical line to Excel chart: scatter plot, bar chart and line graph](https://www.ablebits.com/office-addins-blog/add-vertical-line-excel-chart/)
