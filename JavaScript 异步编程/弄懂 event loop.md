---
title: 弄懂 event loop
date: 2020-10-03
categories:
 - JS
tags:
 - Event Loop
---


## 什么是 event loop

简单来说，event loop 就是 JavaScript 宿主处理事件执行的一种机制。

js 以前是专门用来处理浏览器交互的，比如说 DOM 点击事件等，因此被设计成单线程，所谓单线程，就是同一时间只能处理一件事情，这也就保证了页面中一次只能处理一个事件，避免造成交互混乱的问题。

**开始之前，需要明确的是 JavaScript引擎是单线程的，但是 js 运行环境是多线程。因为浏览器是多线程的，除了 js 引擎线程，还包括 GUI 渲染线程、定时器触发线程、HTTP 请求线程以及 DOM 事件触发线程；node也可以使用 child_process 创建多个子线程。**

现在有事件 A 和 B，如下：

```js
const A = function A() {
    setTimeout(() => {
        console.log('i am A');
    }, 2000)
}

const B = function B() {
    console.log('i am B');
}

A();
B();
```

按照单线程的要求，需要等到 A 执行完毕，才会执行 B，那么打印顺序会是如下所示：

```js
console.log('i am A');
console.log('i am B');
```

但实际上的顺序是：

```js
console.log('i am B');
console.log('i am A');
```

这就是 event loop 机制在起作用，因为 console.log('i am B') 是同步任务，而 setTimeout 是异步任务，同步任务执行完才会去执行异步任务

## 宏任务和微任务

- 什么是宏任务和微任务？

这里有几个概念容易混淆，那就是同步任务和异步任务，宏任务和微任务。js 的代码执行遵循在代码块内从上往下执行的规则，同步任务会依次执行；而异步任务则会分为宏任务和微任务，比如 setTimeout 的第一个参数是宏任务，promise.then 中注册的方法是微任务，会按照宏任务和微任务的执行规则进行执行。

- 宏任务和微任务有哪些？

js 执行的宿主环境有浏览器和 Node,  所以我们通过宿主环境的不同来整理这些异步任务：

| 宏任务                | 浏览器 | node |
| --------------------- | ------ | ---- |
| setTimeout            | √      | √    |
| setInterval           | √      | √    |
| setImmediate          | x      | √    |
| I/O                   | √      | √    |
| requestAnimationFrame | √      | x    |

| 微任务           | 浏览器 | node |
| ---------------- | ------ | ---- |
| mutationObserver | √      | x    |
| promise          | √      | √    |
| process.nextTick | x      | √    |

- 宏任务和微任务的执行顺序

要点一：一个宏任务里可能会包含多个微任务

```js
 new Promise((resolve, reject) => {
     console.log('我是 promise 里的 同步任务')
     resolve('success')
 }).then(res => {
     console.log('我是微任务1')
 }).then(res => {
     console.log('我是微任务2')
 })
 
 setTimeout(() => {
     console.log('我是 setTimeout 里的 宏任务')
 })
```

我们来分析下这段代码里的宏任务和微任务有哪些：
宏任务：setTimeout 的回调
微任务：两个 then 方法的回调

要点二：宏任务是一个一个执行的，而微任务是批量执行的，当前批次微任务没有完成之前，下一个宏任务不会执行

因此执行顺序是:

1. 遇到 promise，参数里面的代码是同步的，所有会先执行 console.log('我是 promise 里的 同步任务')
2. 执行两个 then 方法的回调
3. 执行 setTimeout 的回调

即：

```js
// console.log('我是 promise 里的 同步任务')
// console.log('我是微任务1')
// console.log('我是微任务2')
// console.log('我是 setTimeout 里的 宏任务')
```

## 浏览器下的 event loop

![浏览器 event loop](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/浏览器_event_loop.png)

如图所示，代码执行步骤如下：

1. js 在执行代码时，会将代码放入执行栈中，遇到同步任务，会依次执行；遇到如 DOM 点击事件、ajax 以及 定时器等异步任务，浏览器会交给其他辅线程调用 WebApis  进行处理。
2. 如果 WebApis 处理的异步任务有了结果，就会将该任务推入到回调队列（callback queue）中，回调队列分为宏任务队列和微任务队列。
3. 一旦执行栈内（stack）的任务执行完成，就会将回调队列里的任务放入执行栈中执行，顺序如下：
   1. 如果微任务队列中存在任务，则一次性执行所有微任务
   2. 将宏任务队列的第一个任务放入执行栈中执行

以上步骤的循环就是浏览器中的 event loop。

看一个简单例子感受下：

```js
setTimeout(function callback1(){console.log(3)})

Promise.resolve('success')
.then(function callback2(){console.log(1);return 1})
.then(function callback3(){console.log(2);return 2})

console.log('start')
// start
// 1
// 2
// 3
```

1. 遇到 setTimeout，等待时间到达后将 callback1 放入宏任务队列
2. 遇到 promise.then，等待时间到达后将 callback2 与 callback3  按顺序放入微任务队列
3. 执行同步任务 console.log。
4. 执行完同步任务，将微任务队列里的任务一次性执行
5. 微任务执行完之后，执行宏任务队列的第一个任务

## node 下的 event loop

```js
   ┌───────────────────────────┐                             
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐                                    
│  │     pending callbacks     │                             
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤ connections,  │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```

如上图所示，node 中的 event loop 分为6个循环阶段，当 node.js 启动的时候，会初始化 event loop

- timers： 这个阶段执行 setTimeout 和 setInterval 的回调函数
- pending callbacks：在下一个循环中执行 I/O 回调函数
- idle, prepare：只在 node 内部触发
- poll：获取新的 I/O 事件，在适当的时候阻塞在这里
- check：执行 setImmediate 的回调函数
- close callbacks：执行关闭事件的回调函数，如 socket.on('close', ... )

下面主要对 timers、poll、check 三个阶段进行解析：

### 一、timers

timers 阶段的回调函数可能并不会按照设定的时间延迟去执行，因为 event loop 初始化或者其他阶段回调函数的长时间执行会延迟它们的执行。

```js
const fs = require('fs');

function someAsyncOperation(callback) {
    // 假设读取文件需耗时 95ms
    fs.readFile('./a.txt', callback)
}

cosnt timeoutScheduled = Date.now();

setTimeout(() => {
    const delay = Date.now() - timeoutScheduled;
    console.log(`${delay}ms have passed since I was scheduled`);
}, 100)

someAsyncOperation(() => {
    const startCallbackTime = Date.now();
    while (Date.now() - startCallbackTime < 10) {
        // do something
    }
})
```

上方示例在 node 里的大致执行步骤如下：

1. timer 阶段：因为需要延迟 100ms，所以当前没有 callback 需要执行，进入 pending callback 阶段
2. pending callback阶段：没有 I/O 回调需要执行，进入 idle，
3. idle 忽略
4. poll 阶段：因为此时有 I/O 操作，因此会阻塞在这里，等待95ms至文件读取结束，然后将 callback 放入队列进行执行，耗时10ms。调用结束后，当前队列为空，检查 timers，发现设定时间为95ms，当前运行时间超时了，因此进入 timer 阶段执行回调，所以会打印出"105ms has passed since I was scheduled"

### 二、poll

poll 阶段主要有两个功能：

1.计算它应该阻塞和轮询 I/O 多长时间

2.处理该阶段事件

当 event loop 进入 poll 阶段并且未设定定时器，会出现下面中的某个情况：

- 如果 poll 的队列不为空，那么就会遍历队列，并异步执行完所有回调函数，或者执行耗时达到系统设定的最大时间
- 如果 poll 队列为空，那以下情况中的一个会出现：
    1. 如果代码设定了 setImmediate 方法，event loop 会结束 poll 阶段，进入 check 阶段去执行 check 队列
    2. 如果没有设定 setImmediate 方法，就会阻塞在 poll 阶段，直到有 poll callback 添加到队列中，然后立刻执行。

如果 event loop 进入 poll 阶段并且设定了定时器：

- 一旦 poll 队列处于空闲状态，event loop 会查看 timers 里的回调函数，如果至少有一个回调函数的时间到了，event loop 会按循环顺序进入 timers 阶段去执行这些回调函数。
- 按循环顺序说的是 event loop 不会直接进入 timers 阶段，而是要先进入 check、close callback 之后，再进入 timers 阶段。

### 三、check

- 这个阶段用来存放 setImmediate 回调函数，如果代码中设定了，那么 event loop 不会阻塞等待在 poll 阶段，而是会进入 check 阶段。
- 当 poll 阶段结束，进入check 阶段后，会调用 libuv api 去执行回调函数

### 四、API 比较

- setTimeout 和 setImmediate
    1. setTimeout 设定一个任务在等待指定时间后去执行
    2. setImmediate 在 poll 阶段完成后立即去调用它设定的代码
    3. 它们回调函数执行的顺序依据它们执行的方式会有不同：如果它们的执行不在 I/O 操作里，那么顺序时不定的，如果在 I/O 中，永远都是 setImmediate 最先执行

- process.nextTick

在技术上来说，nextTick 不属于 event loop 的一部分，凡是放进 nextTick 队列的回调函数会在下一次 event loop 循环开始前执行。需要注意的是，正是因为这个特性，如果递归调用 nextTick，会导致下一次 event loop 无法开始。

### node 中宏任务和微任务

因为 node 中的 宏任务分处于不同的阶段，并且微任务中的 process.nextTick 都是先于其它微任务执行，所以可以理解为 有4个宏任务队列以及2个微任务队列。

这里为了便于理解，借用一张图：

<img src="https://user-gold-cdn.xitu.io/2018/9/5/165a8667e6b79cc8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1" alt="node-eventloop" style="zoom:80%;;margin-left:0" />

这里的宏任务和微任务流程模型与浏览器的相同，区别在于：

1. node 中的宏任务队列执行顺序取决于 event loop 所处的阶段
2. 微任务中，process.nextTick 独处一个队列，比其他微任务要早执行

## 总结

event loop 相当于一个总指挥，负责 js 任务的协调与调度。

参考文献 <https://juejin.cn/post/6844903670291628046#heading-5>
