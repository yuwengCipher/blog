---
title: 跟着规范实现一个功能完备的Promise
date: 2020-08-18
categories:
 - JS
tags:
 - Promise
---

一直以来，对于 `promise`，只知道如何使用，其内部的运作机制却不得而知。本着知其然，知其所以然（为了让自己用得安心）的理念，决定跟着规范去了解底层的原理，并手写一个功能完备的 MyPromise.

## 术语

- `promise` 是一个对象或者函数，拥有 `then` 方法
- `thenable` 可以理解为一个拥有 `then` 方法的对象或函数
- `value` 是一个合法的 `JavaScript` 值
- `reason` 用来表示 `promise` 拒绝的原因

## 特点

- `promise` 初始状态为 `pending`，可以转变成 `fulfilled` 或者 `rejected`
- 如果状态是 `fulfilled`，则不能转变为 `rejected` 或者 `pending`。`rejected` 同理。

## 实现

下面开始尝试第一版：

平时都是通过 `new` 来创建一个 `promise` 实例：

```js
const p = new Promise((resolve, reject) => {
    // do something
    resolve('xxx')
})
```

于是首先创建一个 `promise` 构造函数，接收一个方法 `executor` 作为参数, 在内部直接执行，并且传入两个方法以供使用者使用。

```js
function MyPromise(executor) {
    const resolve = () => {}
    const reject = () => {}
    
    executor(resolve, reject);
}
```

如果调用 `resolve` 方法，会将 `Promise` 实例状态转变成 `fulfilled`，如果调用 `reject` 方法，则会将 `Promise` 实例状态转变成 `rejected`。所以接下来给 `MyPromise` 构造函数添加相应属性，并实现 `resolve` 和 `reject`。

```js
function MyPromise(executor) {
    this.status = 'pending';
    this.value = null;
    this.reason = null;
    
    const resolve = (value) => {
    	// 只有处于 pending 状态，才能发生状态改变
        if (this.status === 'pending') {
            this.status = 'fulfilled';
            this.value = value;
        }
        
    }
    const reject = (reason) => {
        if (this.status === 'pending') {
            this.status = 'rejected';
            this.reason = reason;
        }
    }
    
    executor(resolve, reject);
}
```

构造函数建造完毕，现在来处理最主要的部分 `then` 方法，这也是规范给出详细标准的一部分。

```js
// 可以接收两个方法作为参数, 在内部可以根据 MyPromise 实例的状态进行相应的操作
MyPromise.prototype.then = function(onFulfilled, onRejected) {
    
    if (this.status === 'pending') {
        
    }
    
    if (this.status === 'fulfilled') {
        
    }
    
    if (this.status === 'rejected') {
        
    }
}
```

当 `then` 执行的时候，如果 `status` 是 `fulfilled` 或者 `rejected` 状态，可以直接执行 `onFulfilled` 或者 `onRejected` 方法，但如果依然还是 `pending`，需要将这些执行操作放入等待区，也就是存入到回调队列中，如下：

```js
MyPromise.prototype.then = function(onFulfilled, onRejected) {
    if (this.status === 'pending') {
        this.onFulfilledStack.push(() => {
            onFulfilled(this.value);
        })
        this.onRejectedStack.push(() => {
            onRejected(this.reason)
        })
    }

    if (this.status === 'fulfilled') {
        onFulfilled(this.value);
    }
    
    if (this.status === 'rejected') {
        onRejected(this.reason)
    }
}
```

那么现在，待执行栈已经存在了状态改变的回调，需要在合适的时机去执行，所以需要完善 `resolve` 和 `reject` 方法。一旦状态改变，则将带执行栈中的回调全部执行。

```js
const resolve = (value) => {
    this.status = 'fulfilled';
    this.value = value;
    // 执行回调
    while (this.onFulfilledStack.length > 0) {
        this.onFulfilledStack.shift()(this.value);
    }
}
const reject = (reason) => {
    this.status = 'rejected';
    this.reason = reason;
    // 执行回调
    while (this.onRejectedStack.length > 0) {
        this.onRejectedStack.shift()(this.reason);
    }
}
```

我们都知道 `then` 方法必须返回一个 `promise`，因此需要对 `then` 方法进一步改造：

```js
MyPromise.prototype.then = function(onFulfilled, onRejected) {
    // ...
    return new Promise((resolve, reject) => {
        
    })
}
```

到这里就要思考一下，`then` 方法为什么要返回一个 `promise`？ 原因是每一个 `promise` 都会有一个 `then` 方法，而如果 `then` 方法也返回一个 `promise`，那么这个 `then` 也会有一个 `then` 方法，于是可以像下方代码一样链式调用 ：

```js
new MyPromise((resolve, reject) => {
    resolve('success')
}).then().then()
```

但还有个原因。我们不仅可以像上方一样 `resolve` 一个基本值，也可以 `resolve` 一个 `promise`，如下方例子：

```js
new MyPromise((resolve, reject) => {
    resolve(new MyPromise((_resolve, _reject) => {
        setTimeout(() => {
            _resolve('success')
        }, 2000)
    }))
}).then().then()
```

因为被 `resolve` 的 `promise` 的状态是尚未改变的，因此可以将这个 `promise` 放进 `then` 返回的这个 `promise` 内去等待状态改变，所以这一步我们将 `then` 方法内原先的处理逻辑挪到这个返回的 `promise` 内部。

```js
MyPromsise.prototype.then = function(onFulfilled, onRejected) {
    let _promise = null;
    return _promise = new MyPromise((resolve, reject) => {
        // 因为 onFulfilled(this.value) 和 onRejected(this.reason) 可能返回一个 thenable，因此需要将下方代码移入新 promise内部去执行
        if (this.status === 'pending') {
            this.onFulfilledStack.push(() => {
                onFulfilled(this.value);
            })
            this.onRejectedStack.push(() => {
                onRejected(this.reason)
            })
        }
    
        if (this.status === 'fulfilled') {
            onFulfilled(this.value);
        }
        
        if (this.status === 'rejected') {
            onRejected(this.reason)
        }
    })
    
}
```

有个问题，我们在内部直接调用 `onFulfilled` 和 `onRejected`，但却没有对这两个方法类型进行错误处理，也就是必须保证它们是 `function`。

```js
MyPromsise.prototype.then = function(onFulfilled, onRejected) {
    // 设置默认的回调方法（需原样返回传进来的值或者抛出同样的值），可以保证 promise 结果能够透传
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value;
    onRejected = typeof onRejected === 'function' ? onRejected : reason => { throw reason }
    // ...
}
```

这里有一个处理，如果 `onFulfilled` 和 `onRejected` 不是 `function`，那么就将它们赋值成方法，并且将接收到的值进行相应处理：如果是 `onFulfilled`，直接将值 `return`，如果是 `onRejected`, 主动抛出一个错误。这也就实现了 `promise` 值的透传

```js
new MyPromise((resolve, reject) => {
    resolve('success')
}).then().then((value) => {
    console.log(value)  // success
})
```

目前为止，`MyPromise` 已经具备了可实例化，可执行同步任务的功能。但还无法执行异步任务。

规范2.2.4： onFulfilled or onRejected must not be called until the execution context stack contains only platform code。

意思是：`onFulfilled` 和 `onRejected` 方法需要异步执行。

接下来对 `then` 方法进行进一步完善, 将它们的执行丢到异步环境中

```js
MyPromsise.prototype.then = function(onFulfilled, onRejected) {
    // ...
    let _promise = null;
    return _promise = new MyPromise((resolve, reject) => {
        if (this.status === 'pending') {
            setTimeout(() => {
                this.onFulfilledStack.push(() => {
                    onFulfilled(this.value);
                })
                this.onRejectedStack.push(() => {
                    onRejected(this.reason)
                })
            })
        }
    
        if (this.status === 'fulfilled') {
            setTimeout(() => {
                onFulfilled(this.value);
            })
        }
        
        if (this.status === 'rejected') {
            setTimeout(() => {
                onRejected(this.reason)
            })
        }
    })
    
}
```

规范2.2.7.1 If either onFulfilled or onRejected returns a value x, run the Promise Resolution Procedure [[Resolve]](promise2, x)

意思是： 对 `onFulfilled` 或者 `onRejected` 返回的值 x 进行 `resolvePromise` 操作，即需要将 x 当作一个 `thenable` 来对待，`then` 返回的 `promise` 的 状态需要 x 的状态来决定。

这里需要注意： `resolvePromise` 需要 `promise2`（即 then 返回的 promise） 和 x 这两个参，但是 `promise2` 的状态需要它自身 `resolve` 和 `reject` 去改变，因此将 `resolve` 和 reject 也带上。

改动如下：

```js
if (this.status === 'pending') {
    this.onFulfilledStack.push(() => {
        setTimeout(() => {
            let x = onFulfilled(this.value);
            resolvePromise(_promise, x, resolve, reject)
        })
    })
    this.onRejectedStack.push(() => {
        setTimeout(() => {
            let x = onRejected(this.reason);
            resolvePromise(_promise, x, resolve, reject)
        }) 
    })
}

if (this.status === 'fulfilled') {
    setTimeout(() => {
        let x = onFulfilled(this.value);
        resolvePromise(_promise, x, resolve, reject)
    })
}

if (this.status === 'rejected') {
    setTimeout(() => {
        let x = onRejected(this.reason);
        resolvePromise(_promise, x, resolve, reject)
    })
}
```

接下来就是实现 `resolvePromise` 方法了。

按照规范 2.3 The Promise Resolution Procedure，一步步实现：

```js
function resolvePromise(_promise, x, resolve, reject){
    // 2.3.1 如果 _promise 和 x 是同一个对象，reject TypeError
    if (x === _promise) {
        return reject(new TypeError(`${x} should no refer to the same object with MyPromise`))
    }
    
    // 当 x 是对象或者函数时：
    // 判断 x.then 方法中 onFulfilled 回调或者 onRejecetd 回调是否执行过
    // 因为规范规定：其中每一个回调只能执行一次
    // 当其中某项执行过，就将 hasCalled 置为 true
    let hasCalled = false;
    
    if (x instanceof MyPromsie) {
        // 如果状态没有改变，则需要调用 then 方法，然后在内部还需要对以后的返回值进行 resolvePromise 
        if (x.status === 'pending') {
            x.then(y => {
                resolvePromise(_promise, y, resolve, reject)
            }, err => {
                resolvePromise(_promise, err, resolve, reject)
            })
        } else {
            // 如果状态已经改变，那么 x 就会有一个正常值，假设为 z
            // 执行 x.then(resolve, reject)，会直接调用 resolve(z) 或者 reject(z) ：
            // 2.3.2.2 && 2.3.2.3
            x.then(resolve, reject);
        }
    } else if (Object.prototype.toString.call(x) === '[object Object]' || typeof x === 'function') { // x 是对象或者函数
        try {
            let then = x.then;
            if (typeof then === 'function') {
                then.call(x, y => {
                    if (hasCalled) { return }
                    hasCalled = true;
                    resolvePromise(_promise, y, resolve, reject)
                }, err => {
                    if (hasCalled) { return }
                    hasCalled = true;
                    reject(err);
                })
            } else {
                resolve(x)
            }
        } catch(err) {
            // 2.3.3.3.4.1
            // if resolvePromise or rejectPromise have been called, ignore it.
            if (!hasCalled) {
                reject(err)
            }
        }
        
    } else {
        // 2.3.4
        resolve(x);
    }
}
```

如果能通过 `promiseA+` 测试，说明该版本的 `Promise` 符合规范，但是还缺少常用的功能，继续完善：

**MyPromise.resolve**

接收一个值，在内部创建一个新的实例，将状态交给新实例去处理

```js
MyPromise.prototype.resolve = function(value) {
    return new MyPromise((resolve, reject) => {
        resolve(value)
    })
}
```

**MyPromise.catch**

接收一个方法，只会在 `rejected` 状态下执行

```js
MyPromise.prototype.catch = function (callback) {
    return this.then(null, callback);
}
```

**MyPromise.finally**

接收一个方法，不论 `fulfilled` 或者 `rejected` 都会执行

```js
MyPromise.prototype.finally = function (callback) {
    return this.then(callback, callback);
}
```

**MyPromise.all**

接收一个数组，只有所有项的状态为 `fulfilled`，最终结果才为 `fulfilled`，如果有一个 `rejected`，那么结果就是 `rejected`

```js
MyPromise.prototype.all = function (promiseArr) {
    return new MyPromise((resolve, reject) => {
        let result = [];
        let resolveCount = 0;
        promiseArr.forEach((currentPromise, index) => {
            currentPromise.then(value => {
                result[index] = value;
                resolveCount++;
                if (resolveCount === promiseArr.length) {
                    resolve(result);
                }
            }, err => {
                reject(err)
            })
        })

    })
}
```

**MyPromise.race**

接收一个数组，结果由第一个状态改变的 `thenable` 决定

```js
MyPromise.prototype.race = function (promiseArr) {
    return new MyPromise((resolve, reject) => {
        promiseArr.forEach((currentPromise, index) => {
            currentPromise.then(value => {
                resolve(value);
            }, err => {
                reject(err)
            })
        })
    })
}
```

**MyPromise.allSettled**

接收一个数组，只有等到所有项的状态都改变了，不论是 `fulfilled` 还是 `rejected`，都只会变成 `fulfilled`

```js
MyPromise.prototype.allSettled = function(promiseArr) {
    return new Promise((resolve, reject) => {
        let resultArr = [];
        promiseArr.forEach(currentPromise => {
            currentPromise.then(value => {
                resultArr.push({status: 'fulfilled', value: value});
                if (resultArr.length === promiseArr.length) {
                    resolve(resultArr);
                }
            }, err => {
                resultArr.push({status: 'rejected', reason: err});
                if (resultArr.length === promiseArr.length) {
                    resolve(resultArr);
                }
            })
        })
    })
}
```

**MyPromise.any**

接收一个数组，如果其中有一项的状态为 `fulfilled`， 那么结果就是 `fulfilled`，否则如果所有都是 `rejected`，那结果就是 `rejected`， 并且 `reanson` 是 'AggregateError: All promises were rejected'

```js
MyPromise.prototype.any = function(promiseArr) {
    return new Promise((resolve, reject) => {
        let rejectCount = 0;
        promiseArr.forEach(currentPromise => {
            currentPromise.then(value => {
                resolve(value);
            }, err => {
                rejectCount ++;
                if (rejectCount === promiseArr.length) {
                    reject('AggregateError: All promises were rejected');
                }
            })
        })
    })
}
```

测试方法：

\#安装 promises-aplus-tests

```js
npm i promises-aplus-tests -g
```

\#在代码里加上这段

```js
MyPromise.deferred = function () {
    const defer = {};
    defer.promise = new MyPromise((resolve, reject) => {
        defer.resolve = resolve;
        defer.reject = reject;
    })
    return defer;
}
```

## 最后

```js
promises-aplus-tests promise.js
```

有一点需要注意：我在实现 `promise` 内部异步执行时采用的是 `setTimeout`，而 `promise` 的 `then` 方法是一个微任务这与实际有出入。不过用于理解其中的异步理念已经足够了。追求完美的同学可自行实现不同的版本。

完整代码地址：<https://github.com/yuwengCipher/MyPromise>
