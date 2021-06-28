---
title: 浅析 generator 与 async 原理
date: 2021-04-04
categories:
 - 异步编程
tags:
 - Generator
 - Async
---

#### 序言

geneartor 和 async 都是 js 处理异步操作发展历程的产物，它们让异步编程越来越像同步编程。但是由于采用新的语法、关键字，它们是如何做到“类同步”操作的，我们不得而知，因此这篇文章将会窥探内部的秘密。

下面将借助 babel 本地编译示例代码来进行分析。

#### 准备步骤

 一、全局安装 regenerator

```
npm i regenerator -g
```

 二、编译 generator

​		命令加上 --include-runtime，可以得到完整的编译代码

```
regenerator --include-runtime test.js > test-fill.js
```

test-fill.js 就是编译后的代码文件。

#### 解析 generator

现在有如下代码：

```
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

调用 genFn 并不会直接执行，而是会返回一个 generator 对象，每次调用它的 next 方法会执行一个 yield 语句，并返回当前的值及状态。

这是编译后的代码：

```
var _marked = /*#__PURE__*/regeneratorRuntime.mark(genFn);

function genFn () {
	return regeneratorRuntime.wrap(function genFn$ (_context) {
		while (1) {
			switch (_context.prev = _context.next) {
				case 0:
					_context.next = 2;
					return 1;
				case 2:
					_context.next = 4;
					return 2;
				case 4:
					_context.next = 6;
					return 3;
				case 6:
				case "end":
					return _context.stop();
			}
		}
	}, _marked);
}

var g = genFn();
console.log(g.next());
console.log(g.next());
console.log(g.next());
console.log(g.next());
```

可以看到 genFn 被改写了，调用 regeneratorRuntime.wrap 方法创建，其中传给 wrap 方法的 genFn$ 函数，里面的逻辑很简单，是一个永远都会执行的迭代，里面 switch 中的 case 是原代码中 yield 关键字所在的行数，genFn$ 的参数 _context 记录了原代码的执行上下文内容，每次调用 next 方法，实际上就会调用 genFn$，然后执行对应的逻辑。

大致的流程弄清楚之后，再来看看 regeneratorRuntime。

wrap 方法第二个参是 mark(genFn) 的值， mark 方法如下：

```
var $Symbol = typeof Symbol === "function" ? Symbol : {};
var iteratorSymbol = $Symbol.iterator || "@@iterator";

var toStringTagSymbol = $Symbol.toStringTag || "@@toStringTag";

function GeneratorFunctionPrototype () { }

var IteratorPrototype = {};
IteratorPrototype[iteratorSymbol] = function () {
	return this;
};
GeneratorFunctionPrototype.prototype = Object.create(IteratorPrototype);

exports.mark = function (genFun) {
	if (Object.setPrototypeOf) {
		Object.setPrototypeOf(genFun, GeneratorFunctionPrototype);
	} else {
		genFun.__proto__ = GeneratorFunctionPrototype;
		define(genFun, toStringTagSymbol, "GeneratorFunction");
	}
	genFun.prototype = Object.create(Gp);
	return genFun;
};
```

1. IteratorPrototype 对象添加 Symbol.iterator属性，使得 IteratorPrototype 拥有 Iterator 接口
2. GeneratorFunctionPrototype 函数的原型指向 IteratorPrototype ，所以 GeneratorFunctionPrototype 也拥有 Iterator 接口
3. genFun 的   __proto__  指向 GeneratorFunctionPrototype ，同理 genFun  拥有 Iterator 接口
4. mark 返回 genFun

也就是说 mark 方法返回一个拥有 Iterator 接口 genFun。现在看下 wrap 方法。

```
function wrap (innerFn, outerFn, self, tryLocsList) {
	// 确保 protoGenerator 拥有 Itrator 接口
	var protoGenerator = outerFn && outerFn.prototype instanceof Generator ? outerFn : Generator;
	var generator = Object.create(protoGenerator.prototype);
	// context 即 genFn$ 的参数 _context 
	var context = new Context(tryLocsList || []);

	// ._invoke 方法整合了 next、throw 和 return 方法
	generator._invoke = makeInvokeMethod(innerFn, self, context);

	return generator;
}

// _invoke 即内部的 invoke
function makeInvokeMethod (innerFn, self, context) {
    return function invoke (method, arg) {
        // ...
    };
}
```

先看看 _invoke 在哪儿调用

```
var Gp = GeneratorFunctionPrototype.prototype = Generator.prototype

defineIteratorMethods(Gp);
// 为 Generator 添加 prototype 添加 next、throw、return 方法
// 并且这些方法都会调用 _invoke 方法
function defineIteratorMethods (prototype) {
	["next", "throw", "return"].forEach(function (method) {
		define(prototype, method, function (arg) {
			return this._invoke(method, arg);
		});
	});
}
```

这些弄明白之后，可以简单理一下编译后代码的执行步骤：

1. var g = genFn();
   1. genFn() 返回 generator 对象，拥有 _invoke 方法
   2. g 拥有 _invoke
2. g.next();
   1. g 的 next 方法会调用 _invoke 方法
   2. g.next()  ==>  g._invoke();

到这里就可以去看看 invoke 是如何实现的

```

// 四种状态：开始、执行中、暂停执行、结束
var GenStateSuspendedStart = "suspendedStart";
var GenStateExecuting = "executing";
var GenStateSuspendedYield = "suspendedYield";
var GenStateCompleted = "completed";

// generator 方法执行的状态
var state = GenStateSuspendedStart;

return function invoke (method, arg) {
	// 方法体内的语句执行过程中不允许继续执行
    if (state === GenStateExecuting) {
        throw new Error("Generator is already running");
    }

    if (state === GenStateCompleted) {
        if (method === "throw") {
            throw arg;
        }
    }

    context.method = method;
    context.arg = arg;

    while (true) {
        // 省略容错逻辑
        
        state = GenStateExecuting;
		
		// fn.call(obj, arg)
		// fn 即 genFn$，arg 即为 context
		//function tryCatch (fn, obj, arg) {
        //    try {
        //        return { type: "normal", arg: fn.call(obj, arg) };
        //    } catch (err) {
        //        return { type: "throw", arg: err };
        //    }
        //}
       	
       	// 在这里调用 genFn$ 方法
        var record = tryCatch(innerFn, self, context);
        
        if (record.type === "normal") { // 执行 genFn$ 成功会走如下逻辑
        	// 如果语句执行完毕则结束执行，否则暂停执行
        	state = context.done
                    ? GenStateCompleted
                    : GenStateSuspendedYield;
            // 返回执行 yield 的结果
            return {
                value: record.arg,
                done: context.done
            };

        } else if (record.type === "throw") { // 遇到错误则将方法改为 throw，进入下一个循环执行容错逻辑
            state = GenStateCompleted;
            context.method = "throw";
            context.arg = record.arg;
        }
    }
}; 

```

invoke 通过调用的方法进行不同的操作。遇到  throw 直接 throw 错误；遇到 return 会去 complete generator；遇到 next，就会调用 genFn$ 方法，最终返回一个 value 和 done 属性的对象。

#### 解析 async

对于 async 函数，依然使用上面的方法进行编译处理。async 编译后的代码与 generator 编译后的 regeneratorRuntime 对象是一样的，因此我们只需要关注不同点就可以了。

编译前

```
const p = () => new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve(2)
    }, 1000)
})
async function asyncFn() {
    let a = await p();
    let b = await 1;
    let c = await 1;
    console.log(a)
    console.log(b)
    console.log(c)
}

asyncFn();
// 2 1 1
```

编译后

```
var p = function () {
	return new Promise(function (resolve, reject) {
		setTimeout(function () {
			resolve(2);
		}, 1000);
	});
};

function asyncFn () {
	var a, b, c;
	return regeneratorRuntime.async(function asyncFn$ (_context) {
		while (1) {
			switch (_context.prev = _context.next) {
				case 0:
					_context.next = 2;
					return regeneratorRuntime.awrap(p());
				case 2:
					a = _context.sent;
					_context.next = 5;
					return regeneratorRuntime.awrap(1);
				case 5:
					b = _context.sent;
					_context.next = 8;
					return regeneratorRuntime.awrap(1);
				case 8:
					c = _context.sent;
					console.log(a);
					console.log(b);
					console.log(c);
				case 12:
				case "end":
					return _context.stop();
			}
		}
	}, null, null, null, Promise);
}

asyncFn();
```

与 generator 编译之后的代码基本相同，不同的是

1. 调用的是 regeneratorRuntime.async 方法，接收4个参，其中最后一个是 Promise
2. 内部方法 return  regeneratorRuntime.awrap, 即将 await xx 改为 regeneratorRuntime.awrap(xx)

按照顺序，先来看看 async 方法。

```
exports.async = function (innerFn, outerFn, self, tryLocsList, PromiseImpl) {
	// 确保 PromiseImpl 是 Promise
    if (PromiseImpl === void 0) PromiseImpl = Promise;
    
    var iter = new AsyncIterator(
    	// wrap 方法返回一个 generator 对象
        wrap(innerFn, outerFn, self, tryLocsList),
        PromiseImpl
    );
	// iter.next() 是一个 promise 对象
    return exports.isGeneratorFunction(outerFn)
        ? iter
        : iter.next().then(function (result) {   
            return result.done ? result.value : iter.next();
        });
};
```

new AsyncIterator 内部做了些什么工作呢？简化之后就很明了。

```
function AsyncIterator (generator, PromiseImpl) {
    function invoke (method, arg, resolve, reject) {}
    
    function enqueue (method, arg) {
        function callInvokeWithMethodAndArg () {
        	// 返回 promise 实例
            return new PromiseImpl(function (resolve, reject) {
                invoke(method, arg, resolve, reject);
            });
        }
        var previousPromise;
        return previousPromise = callInvokeWithMethodAndArg();
    }
    this._invoke = enqueue;
}
```

defineIteratorMethods 方法将 next(还有 throw、return) 方法代理给了 _invoke，所以 iter.next() 会调用 _invoke，即 enqueue，而 enqueue 返回一个 promise 实例，因此可以调用 then 方法。

```
iter.next().then(function (result) {   
    return result.done ? result.value : iter.next();
})
```

这段是控制 await 顺序执行的开始和结束

首先执行 iter.next()，相当于执行  enqueue，进而执行 invoke。

```
function invoke (method, arg, resolve, reject) {
    var record = tryCatch(generator[method], generator, arg);
    if (record.type === "throw") {
        reject(record.arg);
    } else {
        var result = record.arg;
        var value = result.value;
        if (value &&
            typeof value === "object" &&
            hasOwn.call(value, "__await")) {
            return PromiseImpl.resolve(value.__await).then(function (value) {
                invoke("next", value, resolve, reject);
            }, function (err) {
                invoke("throw", err, resolve, reject);
            });
        }

        return PromiseImpl.resolve(value).then(function (unwrapped) {
            result.value = unwrapped;
            resolve(result);
        }, function (error) {
            return invoke("throw", error, resolve, reject);
        });
    }
}
```

执行步骤如下：

1. var record = tryCatch(generator[method], generator, arg);
   1. 因为 generator[method] 代理给了 _invoke。因此会执行 return { type: "normal", arg: *fn*.call(*obj*, *arg*) };所以这里的 _invoke 是 generator 的 _invoke，而不是 iter 的 _invoke。
   2. 接下来执行 _invoke 的步骤与 generator 函数一样，执行被包裹的函数，最终会返回 {value: xx, done: xx} 对象，但不一样的是，如果函数体没有执行完毕之前，value 是一个对象，有一个 __await 属性。
2. 如果 tryCatch 执行失败，则直接 reject。
3. 如果执行成功：
   1. 如果函数体内的 await 还未执行结束，则会将 record.value.__await 值当作参数，递归调用 invoke 方法
   2. 如果函数体执行完毕，此时的 value 是 undefined，将 record 用异步的方式返回，就会执行 iter.next() 的 then 方法内的回调，这里就是直接执行 return result.value。 因此 async 默认返回 undefined。

到这里，我们就简单理解了 generator 和 async 函数内部的工作原理。其中，async 是在 generator 的基础上工作的，它使用递归方式取代多个 next 方法调用。

奉上一张编译后代码中方法调用的简图以作参考。

![babel 编译 async&generator](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/babel_编译_async_generator.png)

#### 实现一个简版 async

通过上面的分析，我们可以知道，async 就是一个不需要手动执行 next 方法的 generator，明白了这点就好动手了。

先上一个示例

```
function fakeReadFile (filename, duration) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            console.log(filename)
            resolve(filename)
        }, duration)
    })
}

function* genFn () {
    yield fakeReadFile('a.txt', 5000);
    yield fakeReadFile('b.txt', 3000);
    yield 3;
    yield 4;
    yield 5;
}
let g = genFn();
console.log(g.next())
console.log(g.next())
console.log(g.next())
console.log(g.next())
console.log(g.next())
console.log(g.next())

// { value: Promise { <pending> }, done: false }
// { value: Promise { <pending> }, done: false }
// { value: 3, done: false }
// { value: 4, done: false }
// { value: 5, done: false }
// { value: undefined, done: true }
// b.txt   3秒后输出
// a.txt   5秒后输出
```

如上例所示，在实现 async 有2个问题需要解决：

1. 如何保证异步代码按调用顺序去执行（异步代码默认使用 promise 包裹）
2. 如何自动调用 next

先来实现自动调用，首先想到的是声明一个方法，在里面去执行 next 方法，然后递归调用。

```
function asyncFn (genFn) {
    function invokeNext (generator) {
        // 在这里执行 next 方法
        let result = generator.next();
        // 如果调用结束就不再继续调用
        if (result.done === true) {
            return;
        }
        invokeNext(generator)
    }
	// 获取遍历器对象
    let g = genFn.call(null);
    invokeNext(g)
}

// 测试一下
function* genFn_ () {
    yield 1;
    yield 2;
    yield 3;
    yield 4;
    yield 5;
}
// 无异步代码可以按顺序执行
// { value: 1, done: false }
// { value: 2, done: false }
// { value: 3, done: false }
// { value: 4, done: false }
// { value: 5, done: false }
// { value: undefined, done: true }

// 对于前面异步代码的执行就会出先问题
```

js 处理异步操作有多种方式，除了 generator 和 async，我们还有 promise，所以考虑用 promise 来管理每个 next 的调用顺序，事情就变得简单了，完整代码如下：

```
function asyncFn (genFn) {
    function invokeNext (generator) {
        return new Promise((resolve, reject) => {
            let value = generator.next();
            resolve(value)
        }).then((result) => {
            console.log(result);
            if (result.done === true) {
                return;
            }
            // 如果是 promise 对象，则需要在 then 方法回调里去调用下一个 next
            if (result.value instanceof Promise) {
                result.value.then(() => {
                    invokeNext(generator)
                })
            } else {
                invokeNext(generator)
            }
        }, err => {
            reject(err);
        })
    }
    
    const g = genFn.call(null);
    invokeNext(g)
}
```

