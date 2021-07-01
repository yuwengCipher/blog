
## 什么是异步

在 JavaScript 中和生活中都会有异步任务的存在，而异步的产生是有前提条件的，那就是这个任务可以被拆解成两部分。

举个简单的例子。周末我想喝点排骨汤，把所有步骤都准备好后开火炖。正常来说，这锅汤需要3小时才能煲好，很显然我不会傻傻守在这儿等它3小时，因为我还有别的事要做，比如说看电视，嗑瓜子等等。而是3小时后汤煲好了，我才回去盛起来喝。

所以这个喝汤任务由两个步骤组成：煲汤 + 喝汤；这个喝汤步骤就是异步的

## 设计成异步的原因

从上面可以知道，如果我傻傻的在锅前等3个小时，那么我看电视、嗑瓜子等事情就得向后延3小时，到最后我这一天做不了什么，所以傻等3小时完全是浪费时间。

js 是单线程，同一时间只能做一件事，长时间的等待势必会造成资源的浪费

## 异步实现方案

- 回调函数

回调函数在 js 代码里随处可见，如给 DOM 添加点击事件

```js
let d = document.getElementId('test');
d.addEventListener('click', function callback(e) {
    // do something
})
// callback 就是回调函数
```

node 里去读取一份文件

```js
fs.readFile('./test.text', function callback(err, data){
    // do something
})
```

但是有时候的需求会比较复杂，加入多个任务存在依赖性，就会写出如下代码：

```js
fs.readFile('./test.text', function callback(err, data){
    fs.readFile('./test1.text', function callback1(err, data){
        fs.readFile('./test2.text', function callback2(err, data){
            // ...
        })
    })
})
```

深层嵌套让我们很是绝望！

- promise

幸运的是后来有了 promise，带我们逃离了“回调地狱”，来到新世界：

```js
new Promise((resolve, reject) => {
    fs.readFile('./test.text', function callback(err, data){
        resolve(data);
    })
})
.then(data => {
    fs.readFile('./test.text1', function callback1(err, data1){
        // do something
        resolve(data1);
    })
})
.then(data => {
    fs.readFile('./test.text1', function callback2(err, data2){
        // do something
        resolve(data2);
    })
})
```

这样写之后是不是看起来清爽了，瞬间头也不晕了，promise 让流程执行的过程更清晰！

但是你一定见过这样的代码：

```js
new Promise((resolve, reject) => {
    fs.readFile('./test.text', function callback(err, data){
        resolve(data);
    })
})
.then(data => {
    // do something
})
.then(data => {
    // do something
})
.then(data => {
    // do something
})
.then(data => {
    // do something
})
...
```

真的是链式调用一时爽，一直链式调用... 就有点难受了。 promise 抑制了回调地狱的横向扩张，却发现自己的纵向扩张也很厉害。

- generator

generator 函数与普通的函数不同，函数内的代码可以分段执行，也就是说可以暂停执行，凡是需要暂停的地方用 yield 关键字注明。具体用法参考阮老师文章的介绍 <http://www.ruanyifeng.com/blog/2015/04/generator.html>

```js
function* genFn() {
    let x = yield 2;
    let y = yield x * 2;
    return y;
}
let gen = genFn();
gen.next() // {value: 2, done: false}
gen.next() // {value: NaN, done: false}
gen.next() // {value: undefined, done: true}
```

于是对于上面依次读取文件的例子可以改写成如下形式：

```js
// 模拟文件请求
function fakeReadFile(filename, duration) {
    setTimeout(() => {
        console.log(filename)
        // 这里需要主动调用迭代器对象的 next 方法
        gen.next(filename)
    }, duration)
}

function* genFn() {
    // 顺序请求三个文件，期望的是顺序打印出结果
    yield fakeReadFile('a.txt', 5000);
    yield fakeReadFile('b.txt', 3000);
    yield fakeReadFile('c.txt', 1000);
}

const gen = genFn();
gen.next();

// a.txt   5秒后打印
// b.txt   8秒后打印
// c.txt   9秒后打印
```

虽然获取文件所需时长 a > b > c, 但是打印的结果却是 a，b，c，也就是说在 generator 函数内，yield 是按上到下执行的，使得代码执行顺序更清晰明了。

可以看到我需要在文件请求完成之后手动调用 next 方法，因此在实际工作中，我们通常需要将 generator 函数包裹在一个函数内：

```js
function callGen() {
    function* genFn() {
        yield fakeReadFile('a.txt', 5000, gen);
        yield fakeReadFile('b.txt', 3000, gen);
        yield fakeReadFile('c.txt', 1000, gen);
    }
    const gen = genFn();
    gen.next();
}

function fakeReadFile(filename, duration, g) {
    setTimeout(() => {
        console.log(filename)
        // 这里需要主动调用迭代器对象的 next 方法
        g.next(filename)
    }, duration)
}

callGen();
```

总的来说，generator 函数改善了 promise 的 then "链条"过长的缺点，但是需要额外创建一个函数来包装。

- async await

作为 ES7 提出的 async 函数，在 generator 基础上进行优化，是目前 js 处理异步操作的最优解决方案，让异步处理代码可读性更强，流程控制更方便。

我们按照 async 方法的用法来继续优化上面的例子：

```js
function fakeReadFile(filename, duration, g) {
    // 因为 await 关键字后面需要接收一个 promise
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(filename)
        }, duration)
    })
}

async function asyncFn() {
    let a = await fakeReadFile('a.txt', 5000);
    let b = await fakeReadFile('b.txt', 3000);
    let c = await fakeReadFile('c.txt', 1000);
    console.log(a)
    console.log(b)
    console.log(c)
}

asyncFn();

// 9秒后一次性打印
// a.txt 
// b.txt 
// c.txt
```

需要注意的是如果 await 后面表达式里包含异步操作但返回的不是 promise，那么就就不会等待到结果返回，比如这样修改：

```js
function fakeReadFile(filename, duration, g) {
    if (filename === 'a.txt') {
        setTimeout(() => {
            return filename
        }, duration)
    } else {
        return new Promise((resolve, reject) => {
            setTimeout(() => {
                resolve(filename)
            }, duration)
        })
    }
}

async function asyncFn() {
    let a = await fakeReadFile('a.txt', 5000);
    console.log(a)
    let b = await fakeReadFile('b.txt', 3000);
    let c = await fakeReadFile('c.txt', 1000);
    console.log(b)
    console.log(c)
}
asyncFn();

// undefined 立刻输出
// b.txt     4秒之后输出
// c.txt     4秒之后输出
```

也就是说第一个 await 没有阻塞 console.log(a) 的执行。因此 async 对 await 后面的表达式又两种处理方式：

```js
1. promise
   async 会执行表达式并等待有返回值才会继续往下执行代码
2. 非 promise
   async 执行表达式并立刻获取返回值，如果没有则为 undefined
```

## 总结

js 处理异步的方法经历了 回调函数、promise、generator、async 这四个阶段，每一种新方法都是对前方法的改善，主要处理的点有两点：

1. 可读性更强
2. 流程控制更清晰
