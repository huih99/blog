---
title:  HTML中的meta标签你真的了解吗
date: 2021-06-28
tags:
  - html
categories:
  - 笔记
---

# HTML中的meta标签你真的了解了吗

`meta`标签在html里标识的是一些元数据，`meta`标签可以设置哪些元数据，又分别有什么作用呢，接下来一起了解下吧。

## viewport

理解三个重要概念：

1. 布局视口

   布局视口可以理解为文档对象的逻辑尺寸，即css像素（逻辑像素）

2. 理想视口

   理想视口是指布局视口的一个理想尺寸，当布局视口宽度等于设备屏幕宽度(非横向分辨率，而是设备的物理宽度)时，就是理想视口。

3. 视觉视口

   视觉视口指用户正在看到的网页区域，受到浏览器的缩放影响，如果缩放比变小，则用户可以看到更多的网页内容区域，如果缩放比变大，则用户可以看到的网页内容区域变小

   视觉视口宽度 = 理想视口 / 缩放比例

经在chrome浏览器中测试，当设置`<meta name="viewport" content="width=device-width,initial-scale=1.0">`时，当布局视口宽度大于理想视口宽度时，会默认进行缩放，使得布局视口等于理想视口，以使其不出现滚动条。

如果要限制自动缩放，则需要设置为：`<meta name="viewport" content="width=device-width,initial-scale=1.0,minimum-scale=1.0">`，限制最小缩放比例为1。

## 参考链接

[详解meta-viewport标签中的width和initial-scale属性](https://blog.csdn.net/leman314/article/details/111936863)
