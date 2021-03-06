---
title: 实现mobx的autorun函数
date: 2019-05-20
tags:
  - js
categories:
  - 笔记
---

> mobx 是一个数据管理的库，其核心是依赖收集，autoRun 函数就是一个典型的依赖收集

<!-- more -->

## 什么是 autoRun

> autoRun 是一个函数，接受一个函数入参，autoRun 会在第一次调用时执行一次传入的函数，并且收集依赖，而后在函数中使用到的依赖有变动时再次执行

- 示例：
```js
  // autorun的使用
  const obj = observable({ a: 1, b: 2 });
  autoRun(() => {
    console.log(obj.a); //输出1
  });
  obj.b = 3; //什么都不会发生
  obj.a = 2; //这时会再次执行autorun中的函数，打印2

// 为什么？ 因为 autorun 中的函数依赖了 obj.a，所以当 a 有变动时，自热会再次执行函数

```

## 依赖收集

上述代码中的 autoRun 是一个非常智能的函数，能够知道传入的函数到底用到了哪些属性，然后在该属性发生变动时再次执行该函数。 其实这里的过程就是一个依赖收集的过程，vue 的双向绑定的核心就是依赖收集。当函数用到了某个对象或者对象中的某个属性，那么这个对象或者其属性就是这个函数的一个依赖项，可以有很多的依赖，无论哪个依赖发生变动都能检测到并且触发函数再次执行。

## 如何实现依赖收集

在 es5 的时代，实现依赖收集主要是靠对象劫持，通过 Object.defineProperty 定义每一个属性的 getter 与 setter，但是这种方法有缺陷: 一是不能劫持整个对象，只能劫持每个对象属性。二是不能监听到数组变动。在 es6 标准下，有一个新的原生代理函数， Proxy，通过 Proxy 与 Reflect 的组合可以轻松的代理对象，包括数组。这里方便起见，也使用 Proxy 来实现依赖收集。

## 实现思路

在最上面的 mobx 的 autoRun 函数的功能展示中，第一步是定义了一个observable对象，这是最重要的一步，也就是将一个普通对象转为可观察的对象。第二步就是使用 autoRun 来注册 observer(观察者)。所以实现的步骤如下：

1. 实现observable函数
2. 实现autoRun函数

## 开始吧

1. 先定义需要用到的几个变量

```js
//proxies: 储存所有代理对象
const proxies = new WeakMap();
//observers: 储存所有观察函数
const observers = new WeakMap();
//注：使用Map结构是为了存储代理对象时以原对象作为key，使用更方便

/*
 * queuedObservers: observer执行队列
 * 这里采用Set结构是为了去重
 * 防止在当前调用栈中多次添加同一个observer，
 * 如observableData.a = xx; observableData.b = xx
 * 这种情况下observableData.a的observer只应添加一次
 */
let queuedObservers = new Set();
// queued: 当前是否处于observer队列执行中
let queued = false;
// currentObserver: 当前观察者,也就是autoRun执行时的函数
let currentObserver = null;
```

2. 一些辅助函数
```js
/**
 *关联observer到目标对象的key
 *
 * @param {*} target 目标对象
 * @param {*} key 注册observer的key
 */
function registerObserver(target, key) {
  const fns = observes.get(target) || {};
  fns[key] = fns[key] || [];
  fns[key].push(currentObserver);
  observes.set(target, fns);
}

/**
 *队列中插入observer
 *
 * @param {*} target 观察对象
 * @param {*} key key
 */
function queueObservers(target, key) {
  // 取得当前target注册过的所有observers
  const registerObservers = observes.get(target);
  if (registerObservers && registerObservers[key]) {
    // 当前队列中存在的observers
    const existObservers = Array.from(queuedObservers);
    queuedObservers = new Set(existObservers.concat(registerObservers[key]));
    invokeObserves();
  }
}

/**
 *执行队列中的所有observer
 *
 * @returns
 */
function invokeObserves() {
  if (queued) {
    return;
  }

  queued = true;

  // 在下一个事件循环里取出所有队列中的所有observer并执行
  setTimeout(() => {
    for (let observer of queuedObservers) {
      observer();
    }
    queued = false;
    queuedObservers.clear();
  }, 0);
}
```

3. observable 和 autoRun
```js
//observable函数
function observable(obj) {
  return proxies.get(obj) || toObservable(obj);
}

function toObservable(obj) {
  const proxyObj = new Proxy(obj, {
    get(target, key, receiver) {
      const result = Reflect.get(target, key, receiver);
      const resultIsObject = result && typeof result === 'object';
      if (currentObserver) {
        registerObserver(target, key);
      }
      return resultIsObject ? observable(result) : result;
    },
    set(target, key, value, receiver) {
      if (key === 'length' || value !== Reflect.get(target, key, receiver)) {
        queueObservers(target, key);
      }
      return Reflect.set(target, key, value, receiver);
    }
  });
}

//autoRun
function autoRun(fn) {
  currentObserver = fn;
  fn();
  currentObserver = null;
}
```

4. 测试
```js
const observableData = observable({
  name: 'xiaoming',
  age: 18,
  sex: 'male',
  hobbies: ['basketball', 'swim']
});

autoRun(() => {
    console.log(observableData.name) // xiaoming
});

autoRun(() => {
    console.log(observableData.hobbies) // ['basketball', 'swim']
})

observableData.name = 'xiaohe';
observableData.name = '小红';
observableData.hobbies.push('music');

//监测到name和hobbies属性变动
//打印结果： '小红', ['basketball','swim','music'];
```

## 总结

主要是了解依赖收集到底是怎么一回事，无非就是通过属性劫持或者通过es6的proxy代理来达到目的。Proxy能代理的不止get和set，是一个功能很强大的函数，现在Vue3.0版本中也将使用Proxy来代替之前的Object.defineProperty。