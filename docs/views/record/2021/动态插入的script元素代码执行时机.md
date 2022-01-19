---
title:  通过js动态插入的script元素的代码执行时机
date: 2021-12-20
tags:
  - html
categories:
  - 笔记
---

# 通过js动态插入的script元素的代码执行时机

先说结论：

> 如果是在加载页面的过程中通过js动态插入的script元素，无论其元素位置在哪儿，都会在其资源加载完毕后立即执行（如果是直接写在script标签内的代码文本，则会在添加到document中时立即执行，如果通过src加载远程资源， 其资源请求顺序会被放在所有静态dom节点的资源请求顺序之后，并在资源下载完毕后立即执行），同时不会阻塞html的解析过程

## 证明1

demo/1.js

动态的插入script标签加载 demo/3.js

```js
console.log('1 loaded');
const script = document.createElement('script');
script.src = 'demo/3.js';
script.type = 'text/javascript';
document.head.appendChild(script);  // 插入到head元素的尾部
```

demo/2.js

```js
console.log('2 loaded');
```

demo/3.js

```js
const p = document.querySelector('p')
console.log(p) // 打印p元素
console.log('3 loaded');
```

html:

注： http://127.0.0.1:8090会延迟3秒然后响应，并输出 **server loaded**，这里将其称作延时script。

```html
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <script src="demo/1.js"></script>
    <script src="http://127.0.0.1:8090"></script>
    <title>Document</title>
  </head>
  <body>
    <p>其实我很好</p>
    <script src="demo/2.js"></script>
  </body>
</html>
```

 输出如下：

![image-20220111162434969](assets/动态插入的script元素代码执行时机/image-20220111162434969.png)

network面板如下：

![image-20220111144619733](assets/动态插入的script元素代码执行时机/image-20220111144619733.png)

可以看出，即便将script元素添加到尾部，且上面的延时script还未加载完毕，这个动态添加的script元素也在其资源加载完毕后便执行了代码。而且因为此时延时script阻塞了dom解析构建，所以未能找到p元素。

## 证明2

将html稍微改动下，http://127.0.0.1:8090服务也取消延时，改为立即响应：

```html
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <script src="demo/1.js"></script>
    <script src="demo/2.js"></script>
    <title>Document</title>
  </head>
  <body>
    <p>其实我很好</p>
    <script src="http://127.0.0.1:8090"></script>
  </body>
</html>
```

输出如下：

![image-20220111161023546](assets/动态插入的script元素代码执行时机/image-20220111161023546.png)

network面板：

![image-20220111150424892](assets/动态插入的script元素代码执行时机/image-20220111150424892.png)

可以看出，在执行3.js的时候已经完成了首次渲染，因为已经可以打印出p元素，所以动态加载的script标签并不会阻塞dom构建。

## 证明3

如果使用内联代码又是怎样的顺序呢？

将 demo/1.js 代码修改, 动态添加的script标签直接使用内联代码

```js
console.log('1 loaded');
const script = document.createElement('script');
script.innerText="console.log(document.querySelector('p'));console.log('3 loaded')";
script.type = 'text/javascript';
document.head.appendChild(script); // 添加到head的尾部
```

html:

```html
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <script src="demo/1.js"></script>
    <script>
        console.log('2 loaded')
      </script>
    <title>Document</title>
  </head>
  <body>
    <p>其实我很好</p>
  </body>
</html>
```



输出：

![image-20220111161832383](assets/动态插入的script元素代码执行时机/image-20220111161832383.png)

可以看到， **3 loaded** 输出在 **2 loaded**之前，说明script元素被添加到head元素中就立即执行了。
