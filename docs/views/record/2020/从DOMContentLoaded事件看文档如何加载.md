---
title:  从DOMContentLoaded事件看文档如何加载
date: 2020-05-19
tags:
  - css
categories:
  - 笔记
---


# 从DOMContentLoaded事件看文档如何加载

对于DOMContentLoaded事件，网上能搜到的关于这个事件解释如下：

> MDN中的解释：当初始的 HTML 文档被完全加载和解析完成之后，DOMContentLoaded 事件被触发，而无需等待样式表、图像和子框架的完成加载。

意思就是当HTML文档加载解析完毕之后就会执行，但HTML文档到底何时才算作加载解析完毕，DOMContentLoaded具体执行时机如何，从MDN的解释来看依然搞不明白。

所谓的解析就是指浏览器把HTML文本内容解析为DOM树结构，文本中的标签会解析为对应的DOM节点。但是在HTML文档中不只有DOM结构文本，还可能会有`link`和`script`标签加载css和script脚本。这两个标签对文档解析渲染的顺序起到了非常重要的影响。

我们都知道这两个概念：

* css加载时不会阻塞DOM树构建解析，但会阻塞DOM树的渲染。
* js加载时会阻塞DOM树的解析和渲染。

那么具体加载css或js时是如何阻塞的，下面具体实测（测试环境： win10, chrome: 81）。

> 通过本地服务器设置的响应延时进行测试，已知请求index.js延迟4s后响应，请求index.css时服务器会延迟5s后响应。

## css加载时如何阻塞页面的

### css加载是否会阻塞Dom树解析和渲染

#### head标签中使用link加载css

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <script>
        setTimeout(() => {
            const div = document.querySelectorAll('div');
            console.log(div);
        }, 0)
        document.addEventLister('DOMContentLoaded',() => {
            console.log('DOMContentLoaded')
        })
    </script>
    <link rel="stylesheet" href="http://127.0.0.1:9999/index.css">
    <title>Document</title>
</head>

<body>
    <div>div 1</div>
    <div>div 2</div>
</body>

</html>
```

上述代码实际运行后，控制台输出：
1. DOMContentLoaded
2. [div,div]

此时页面未渲染，依然为空白，css加载完成后，才看到页面。

结论如下：
1. **当link标签在head标签内时，css加载时阻塞后续DOM树的渲染，但不阻塞后续DOM结构的解析。**
2. **当link标签在head标签内时，DOMContentLoaded事件不受css加载的影响。**

#### body中使用link标签加载css
  ```html
  <!DOCTYPE html>
  <html lang="en">

  <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <script>
          setTimeout(() => {
              const divs = document.querySelectorAll('div');
              console.log('div节点',divs, `长度${divs.length}`);
          },0)
          document.addEventLister('DOMContentLoaded', () => {
              console.log('DOMContent loaded');
          })
      </script>
      <title>Document</title>
  </head>

  <body>
      <div>before css</div>
      <link rel="stylesheet" href="http://127.0.0.1:9999/index.css">
      <div>after css loaded</div>
  </body>

  </html>
  ```

  上述代码实际运行后，打印结果为：<br>
  1. div节点，[div], 长度1

  2. index.css加载完成后才打印出DOMContent loaded

在index.css还未加载完成时，页面已经渲染出了第一个div节点（页面可见）。

  所以结论如下：
  1. **body标签内使用link标签不仅会阻塞link标签之后的DOM结构的渲染也会影响后续DOM结构的解析**
  2. **body标签内使用link标签会影响到DOMContentLoaded事件的触发，在css资源加载完毕后才触发DOMContentLoaded事件**
  3. **在body标签内遇到link标签时，如果此资源还未加载完成，浏览器会进行首次渲染，将link标签之前的DOM树和CSSOM树合并成一颗Render树,渲染到页面中**


### script加载是否会阻塞DOM树解析和渲染

#### head标签内使用script标签

```html
  <!DOCTYPE html>
  <html lang="en">

  <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <script>
        document.addEventLister('DOMContentLoaded', () => {
            console.log('DOMContentLoaded')
        })
        setTimeout(() => {
            console.log('nodes:',document.querySelectorAll('div'));
        })
      </script>
      <script src="http://127.0.0.1:9999/index.js"></script>
      <title>Document</title>
  </head>

  <body>
      <div>i am here</div>
  </body>

  </html>
```

控制台输出:

1. `nodes:[]`
2. js资源加载执行完毕后输出: `DOMContentLoaded`

当script资源加载执行完毕后，浏览器进行首次渲染

结论：

1. **head标签中使用script标签加载资源会阻塞DOM树的构建和渲染**
2. **DOMContentLoaded事件会在js资源下载执行完毕后触发**

#### body内使用script标签

```html
  <!DOCTYPE html>
  <html lang="en">

  <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <script>
        document.addEventLister('DOMContentLoaded', () => {
            console.log('DOMContentLoaded')
        })
        setTimeout(() => {
            console.log('nodes:',document.querySelectorAll('div'));
        })
      </script>
      <title>Document</title>
  </head>

  <body>
      <div>before script</div>
      <script src="http://127.0.0.1:9999/index.js"></script>
      <div>after script</div>
  </body>

  </html>
```

控制台输出：

1. `nodes: [div]` (说明已经渲染出了第一个节点)
2. index.js资源下载执行完毕后打印`DOMContentLoaded`

index.js资源下载并执行完毕后触发DOMContentLoaded事件，并渲染出第二个div节点。

结论：

1. **在body内script标签会阻塞后续DOM树的构建解析**
2. **DOMContentLoaded事件会在js资源加载执行完毕后触发**
3. **在 body 中第一个 script 资源下载完成之前，浏览器会进行首次渲染，将该 script 标签前面的 DOM 树和 CSSOM 合并成一棵 Render 树，渲染到页面中。这是页面从白屏到首次渲染的时间节点，比较关键。**

### css加载是否会阻塞js的执行

示例1
```html
  <!DOCTYPE html>
  <html lang="en">

  <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <script>
        var t = performance.now();
        document.addEventListener('DOMContentLoaded',() => {
            console.log('DOMContentLoaded')
        })
      </script>
      <link rel="stylesheet" src="http://127.0.0.1:9999/index.css"></link>
      <script>
        console.log('间隔' + (performance.now - t) + 'ms');
      </script>
      <title>Document</title>
  </head>

  <body>
      <div>loaded</div>
  </body>

  </html>
```

控制台输出：

1. `间隔4832.039999997505ms`
2. `DOMContentLoaded`

说明当4832.039999997505ms之后才执行link标签下的脚本代码。所以link标签加载css资源会阻塞后续脚本的执行

因为link标签阻塞了script内代码执行，而script标签又阻塞后续DOM树的解析和渲染， 所以DOMContentLoaded事件其实也会在等待`index.css`资源加载完毕后，script代码执行完毕且DOM结构解析完成后触发事件。

结论

1.**css资源加载会阻塞后续代码的执行**

示例2：
```html
  <!DOCTYPE html>
  <html lang="en">

  <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <script>
        var t = performance.now();
      </script>
      <link rel="stylesheet" src="http://127.0.0.1:9999/index.css"></link>
      <script src="http://127.0.0.1:9999/index.js"></script>
      <title>Document</title>
  </head>

  <body>
      <div>loaded</div>
  </body>

  </html>
```
控制台中network面板中同时发出了`index.js`和`index.css`的资源请求。`index.js`资源响应时间为4s，`index.css`响应时间为5s。但是index.js加载完成仍需等待index.css加载完后才会执行代码。

### script中异步代码是否会阻塞DOMContentLoaded

结论： 不会影响，DOMContentLoaded事件会在所有脚本的同步代码执行完且DOM树解析完成后触发事件