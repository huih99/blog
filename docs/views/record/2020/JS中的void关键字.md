---
title:  JS中的void关键字
date: 2020-08-10
tags:
  - js
categories:
  - 笔记
---

# js中的void

经常在各种开源库中发现void关键字，一直没有搞懂这个void到底起什么作用

> void 在JavaScript中是一个关键字， 返回值永远是undefined。

不论 void 后跟何种类型数据，返回值永远是undefined。

```js
console.log(void 0);
// undefined
console.log(void 'javascript')
// undefined
console.log(void [])
// undefined
console.log(void {})
// undefined
console.log(void function(){})
// undefined
```

为什么要使用`void`？

之所以这么多开源库中大量使用void，其目的就是为了JavaScript中的`undefined`，因为`undefined`在js中并不是关键字，可以使用**undefined**做为变量名，并修改值，为了保证获得原始的`undefined`，遂使用void来获取`undefined`

```js
var undefined = 2;
console.log(undefined); // 2
console.log(void 0 === undefined) // false
console.log(null == undefined) // false
```