---
title:  webpack之publicPath
date: 2021-05-28
tags:
  - webpack
categories:
  - 笔记
---

# webpack之publicPath

## 什么是publicPath

> publicPath: 通过它来指定所有资源的基础路径

发送到 `output.path` 目录的每个文件，都将从 `output.publicPath` 位置引用。这也包括（通过 [代码分离](https://webpack.docschina.org/guides/code-splitting/) 创建的）子 chunk 和作为依赖图一部分的所有其他资源（例如 image, font 等），某些**loader**有`publicPath`选项，可以覆盖此处定义的`publicPath`

例：

```js
module.exports = {
    output: {
        path: 'dist',
        // 指定public path
        publicPath: '/module/sub',
        filename: "js/[name].[contenthash:8].js",
        chunkFilename: "js/[name].[contenthash:8].js"
    }
}
```

通过**html-webpack-plugin**自动生成html如下：

```html
<!DOCTYPE html>
<html lang="">

<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <link rel="icon" href="/module/sub/favicon.ico">
    <title>sub-micro-app</title>
    <link href="/module/sub/css/app.59fb9246.css" rel="preload" as="style">
    <link href="/module/sub/js/app.6aa9ef33.js" rel="preload" as="script">
    <link href="/module/sub/js/chunk-vendors.547e0756.js" rel="preload" as="script">
    <link href="/module/sub/css/app.59fb9246.css" rel="stylesheet">
</head>

<body><noscript><strong>We're sorry but sub-micro-app doesn't work properly without JavaScript enabled. Please enable it
            to continue.</strong></noscript>
    <div id="app"></div>
    <script src="/module/sub/js/chunk-vendors.547e0756.js"></script>
    <script src="/module/sub/js/app.6aa9ef33.js"></script>
</body>

</html>
```

可以看到，所有资源的路径都加上了`publicPath`这段基础路径

`webpack-dev-server`启动后的访问地址也受`publicPath`影响，如果`publicPath`是有效的绝对路径或者相对路径，则开发服务启动后访问的地址为`[host]:[port]/[publicPath]`


## 在运行时设置publicPath

webpack提供了**\_\_webpack_public_path\_\_**这个公共变量，可以在运行时修改引入资源的publicPath

例如，在vue中通过img引入了一张图片，该图片最终会经过file-loader处理

此时，将webpack配置中的publicPath改为`www.baidu.com`

webpack.config.js

```js
module.exports = {
    output: {
        publicPath: "www.baidu.com"
    }
}
```



```vue
// example.vue
<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png" />
    <HelloWorld  msg="Welcome to Your Vue.js App" />
  </div>
</template>
```

将此文件打包后输出的代码为：

```js
// 只截取了创造img元素这部分代码
// 可以看到 src 是一个动态值 r('cf05),就是取cf05这个模块输出的值
n('img', { attrs: { alt: 'Vue logo', src: r('cf05') } })

...

// cf05输出
cf05: function(e, t, r) {
      e.exports = r.p + 'img/logo.82b9c7a5.png';
    }

// r.p其实就是__webpack_require__.p这个变量

// 可以看到__webpack_require__.p被设置为了www.baidu.com

/******/ 	// __webpack_public_path__
/******/ 	__webpack_require__.p = "www.baidu.com/";
/******/
```

可以通过在运行时代码中修改**\__webpack_public_path__**这个变量以动态修改这个值：

```js
__webpack_public_path__ = 'www.google.com';
```

注意**\__webpack_public_path__**不是window上的全局变量，而是一个标记值，在打包的时候这段代码会被替换为

```js
__webpack_require__.p = "www.google.com/";
```



> **warning**
>
> 如果entry文件中使用的是es2015 module import，则会在import之后对__webpack_public_path赋值。在这种情况下，你必须将 public path 赋值移至一个专用模块中，然后将它的 import 语句放置到 entry.js 最上面：

```js
// publicPath.js
__webpack_public_path = "/base"
```

entry js:

```js
// main.js
import './publicPath'
import 'others'
...
```