---
title:  js相等比较算法
date: 2021-06-01
tags:
  - js
categories:
  - 笔记
---

# js相等比较算法

## 1.严格相等 （===）

比较 x === y:

[参考链接](https://262.ecma-international.org/5.1/#sec-11.9.6)

1. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is different from [Type](https://262.ecma-international.org/5.1/#sec-8)(*y*), return **false**
2. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Undefined, return **true**
3. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Null, return **true**
4. If  [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Number, then
   1. If *x* is **NaN**, return **false**
   2. If *y* is **NaN**, return **false**
   3. If *x* is the same Number value as *y*, return **true**
   4. If *x* is **+0** and *y* is **−0**, return **true**
   5. If *x* is **−0** and *y* is **+0**, return **true**
   6. Return **false**
5. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is String, then return **true** if *x* and *y* are exactly the same sequence of characters (same length and same characters in corresponding positions); otherwise, return **false**
6. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Boolean, return **true** if *x* and *y* are both **true** or both **false**; otherwise, return **false**
7. Return **true** if *x* and *y* refer to the same object. Otherwise, return **false**

## 2. 非严格相等 （==）

比较 x == y:

1. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is the same as [Type](https://262.ecma-international.org/5.1/#sec-8)(*y*) , then
   1. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Undefined, return **true**.
   2. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Null, return **true**.
   3. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Number, then
      1. If *x* is **NaN**, return **false**.
      2. If *y* is **NaN**, return **false**.
      3. If *x* is the same Number value as *y*, return **true**.
      4. If *x* is **+0** and *y* is **−0**, return **true**.
      5. If *x* is **−0** and *y* is **+0**, return **true**.
      6. Return **false**.
   4. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is String, then return **true** if *x* and *y* are exactly the same sequence of characters (same length and same characters in corresponding positions). Otherwise, return **false**.
   5. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Boolean, return **true** if *x* and *y* are both **true** or both **false**. Otherwise, return **false**.
   6. Return **true** if *x* and *y* refer to the same object. Otherwise, return **false**.
2. If *x* is **null** and *y* is **undefined**, return **true**.
3. If *x* is **undefined** and *y* is **null**, return **true**.
4. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Number and [Type](https://262.ecma-international.org/5.1/#sec-8)(*y*) is String,
   return the result of the comparison *x* == [ToNumber](https://262.ecma-international.org/5.1/#sec-9.3)(*y*).
5. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is String and [Type](https://262.ecma-international.org/5.1/#sec-8)(*y*) is Number,
   return the result of the comparison [ToNumber](https://262.ecma-international.org/5.1/#sec-9.3)(*x*) == *y*.
6. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Boolean, return the result of the comparison [ToNumber](https://262.ecma-international.org/5.1/#sec-9.3)(*x*) == *y*.
7. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*y*) is Boolean, return the result of the comparison *x* == [ToNumber](https://262.ecma-international.org/5.1/#sec-9.3)(*y*).
8. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is either String or Number and [Type](https://262.ecma-international.org/5.1/#sec-8)(*y*) is Object,
   return the result of the comparison *x* == [ToPrimitive](https://262.ecma-international.org/5.1/#sec-9.1)(*y*).
9. If [Type](https://262.ecma-international.org/5.1/#sec-8)(*x*) is Object and [Type](https://262.ecma-international.org/5.1/#sec-8)(*y*) is either String or Number,
   return the result of the comparison [ToPrimitive](https://262.ecma-international.org/5.1/#sec-9.1)(*x*) == *y*.
10. Return **false**.

这个算法中有个比较重要的求值算法 **toPrimitive**，这是一个内部方法，用来计算对象类型的原始值，也就是将**object**类型转换为非**object**类型，用法：`toPrimitive(x,prefferedType?)`，其中`x`为输入值，`prefferedType`为可选参数，表明想转换为的类型（string 或者 number）,在不传递此参数情况下，默认认为是需要转出number类型值。

求值过程如下：

1. prefferedType ?= number
   1. 尝试调用`x.valueOf`方法,如果该方法存在，且结果值是一个原始值，则返回该值
   2. 尝试调用`x.toString`方法，如果该方法存在，且结果值是一个原始值，则返回该值
   3. 抛出`TypeError`异常
2. prefferedType = string
   1. 尝试调用`x.toString`方法,如果该方法存在，且结果值是一个原始值，则返回该值
   2. 尝试调用`x.valueOf`方法，如果该方法存在，且结果值是一个原始值，则返回该值
   3. 抛出`TypeError`异常

## 3.同值相等（Object.is）

`Object.is(x, y)`比较两值是否相同

规则如下：

1. x, y类型都为`undefined`，true
2. x, y类型都为`null`，true
3. x, y类型都为`string`，且长度和字符以及顺序一致，true
4. x, y类型都为引用类型，并且引用都是内存中同一个对象，true
5. x, y类型都为number
   1. x, y都为 +0，true
   2. x, y都为 -0，true
   3. x, y都为 NaN, true
   4. x, y都为非零和非NaN的值，且值相同， true
6. false
