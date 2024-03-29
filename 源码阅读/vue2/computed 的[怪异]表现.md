---
title: computed 的[怪异]表现
date: 2021-05-12
categories:
 - Vue
tags:
 - Vue
---

## 前言

官方文档中说明了 computed 存在的理由，有两条，一是如果插值表达式中的逻辑比较复杂，则可以使用 computed 代替；二是用来封装复杂计算，因为 computed 拥有缓存功能，只有响应式依赖项发生了改变， computed 的值才会改变。个人觉得第一条使用方法也可以代替，主要原因还是第二条，作用很大。但是不知道大家在使用 computed 属性的时候有没有如下疑问：
1. computed 其实是一个方法，为什么可以像普通值一样使用，不需要带 () 进行调用
2. 依赖项的改变如何促使 computed 改变
3. 为什么 computed 拥有缓存功能
4. 如果没有在模板中使用，即使依赖项发生改变，computed 也不会重新求值
5. 如果依赖项不是响应式的，为什么不会重新求值

刚开始使用的时候，就非常不理解这些不寻常的表现，于是想着通过阅读源码去弄懂这一切，特此记录一下。本文会以官方 demo 为例，

## Vue 如何处理 computed

### computed 为什么可以像普通值一样使用

```js
<div id="demo">
	<p>Computed reversed message: "{{ reversedMessage }}"</p>
</div>

data: {
	message: 'Hello'
},
computed: {
	reversedMessage: function () {
		return this.message.split('').reverse().join('')
	}
}
```

从源码中找到 initState，它是合并完 options 之后执行的方法，里面会调用 initComputed(vm, opts.computed) 处理 computed，opts 就是合并完成的 options。

```js
function initComputed (vm, computed) {
	for (var key in computed) {
		var userDef = computed[key];
		var getter = typeof userDef === 'function' ? userDef : userDef.get;
		// 省略
		if (!(key in vm)) {
			defineComputed(vm, key, userDef);
		}
	}
}
```

遍历 computed 对象，获取到 key 对应的方法，如果是浏览器端，就创建一个 watcher。然后判断 vm 是否已经存在该 key 的属性，如果没有就去定义这个 computed 属性。

```js
function defineComputed (target, key, userDef) {
	var shouldCache = !isServerRendering();
	if (typeof userDef === 'function') {
		sharedPropertyDefinition.get = shouldCache
			? createComputedGetter(key)
			: createGetterInvoker(userDef);
		sharedPropertyDefinition.set = noop;
	}
	// 省略
	Object.defineProperty(target, key, sharedPropertyDefinition);
}
```

sharedPropertyDefinition 拥有 get 和 set 方法，set 是 noop，可以忽视，get 分为两种情况：如果不是服务端渲染，那么就是使用 createComputedGetter('reversedMessage') 创建，否则就是 createGetterInvoker(userDef) 创建。最后使用 Object.defineProperty 将 reversedMessage 这个属性添加到 vm 中。准备工作已经做好了，因为 computed 是放在模板中的，那么肯定会经过 render 函数去处理，下面就是 render 函数。

```js
(function anonymous(
) {
with(this){return _c('div',{attrs:{"id":"demo"}},[_c('p',[_v("Computed reversed message: \""+_s("olleH")+"\"")])])}
})
```

可以看到 {{ reversedMessage }} 被转换成了 _s(reversedMessage)，当 _s(reversedMessage) 执行的时候，相当于执行 _s(this.reversedMessage)，此时 this.reversedMessage 就会触发前面绑定的 get 方法，也就是 computedGetter。

```js
return function computedGetter () {
	var watcher = this._computedWatchers && this._computedWatchers[key];
	if (watcher) {
		if (watcher.dirty) {
			watcher.evaluate();
		}
		if (Dep.target) {
			watcher.depend();
		}
		return watcher.value
	}
}
```

因为是浏览器端，所以在 initComputed 内部声明过 watcher，即：

```js
var computedWatcherOptions = { lazy: true };
if (!isSSR) {
	watchers[key] = new Watcher(
		vm,
		getter || noop,
		noop,
		computedWatcherOptions
	);
}
```

并且 watcher options 的 lazy 设置为 true，在 Watcher 构造函数内会将 dirty 设置成这样: this.dirty = this.lazy，因此 this.dirty = true。也就会执行 watcher.evaluate()。watcher.evaluate 方法又会涉及到 watcher.get 方法，因此同时展示如下：

```js
Watcher.prototype.evaluate = function evaluate () {
	this.value = this.get();
	this.dirty = false;
};

Watcher.prototype.get = function get () {
	pushTarget(this);
	var value;
	var vm = this.vm;
	try {
		value = this.getter.call(vm, vm);
	} catch (e) {
		if (this.user) {
			handleError(e, vm, ("getter for watcher \"" + (this.expression) + "\""));
		} else {
			throw e
		}
	} finally {
		if (this.deep) {
			traverse(value);
		}
		popTarget();
		this.cleanupDeps();
	}
	return value
};
```

首先调用 watcehr.get()，会触发 this.getter.call(vm, vm)，即调用 reversedMessage 方法，将 watcher.value 赋值为 'olleH'; 最后返回 watcher.value 即 'olleH'。那么 render 函数的这部分 [_v("Computed reversed message: \""+_s(reversedMessage)+"\"")]，就变成了 [_v("Computed reversed message: \""+_s("olleH")+"\"")]。这样就实现了 computed 不带()进行使用了，其实是在内部去调用 reversedMessage 方法。这个实现有两个关键点：
1. 将 computed.reversedMessage 代理给了 vm.rereversedMessage
2. 使用 object.defineProperty 对 vm.rereversedMessage 进行数据劫持

### computed 如何响应式改变

在 demo 中 mounted 钩子中加上下面这段代码

```js
setTimeout(() => {
	this.message = 'cipher'
}, 1000)
```

然后可以看到，在一秒后改变 message 的值，页面显示从 Computed reversed message: "olleH" 变成了 Computed reversed message: "rehpic"，符合预期，响应式依赖项改变时，computed 也发生了改变。我们来看看这是怎么实现的。

message 作为一个响应式属性，在最开始进行响应式设定的时候，它拥有一个 get 方法：

```js
var dep = new Dep();
get: function reactiveGetter () {
	// 获取 message 当前值
	var value = getter ? getter.call(obj) : val;
	// 如果收集者存在当前目标
	if (Dep.target) {
		// 将 Dep.target 存入 dep.subs 中
		dep.depend();
		// 如果 observe 返回的值需要响应式处理
		if (childOb) {
			// // 将 Dep.target 存入 childOb.dep.subs 中
			childOb.dep.depend();
			// 处理数组情况
			if (Array.isArray(value)) {
				dependArray(value);
			}
		}
	}
	return value
},
```

上面已经说过，reversedMessage 渲染时会在内部调用 reversedMessage 方法去获取最新的值，而这个值就是 this.message.split('').reverse().join('') 返回的，此时就会触发 message 的 get 方法，即上面那段代码。reversedMessage get 方法触发的时候，会执行 pushTarget(this) 将 Dep.target 设置成 this._computedWatchers['reversedMessage']，那么当执行 dep.depend() 时，最后就会执行 dep.addSub(watcher) 将 watcher 存入 dep.subs 数组中，这个 dep 是 message 的监听收集器，也就是说 message 改变会通知到 reversedMessage watcher。

综合上面说的步骤，总结一下方法执行图：

![computed依赖收集](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/computed依赖收集.png)

执行完 watcher.evaluate，reversedMessage watcher 就被收集到 message 的订阅者集合中了。

1秒之后，this.message 变成 cipher，那么就会去执行 message.set。

```js
set: function reactiveSetter (newVal) {
	// 省略
	dep.notify();
}

Dep.prototype.notify = function notify () {
	// subs 中拥有 reversedMessage
	var subs = this.subs.slice();
	// 如果不是异步的，那么就要将 watcher 按照初始化的顺序进行排序。
	if (!config.async) {
		subs.sort(function (a, b) { return a.id - b.id; });
	}
	// 遍历订阅者，执行订阅者的更新操作
	for (var i = 0, l = subs.length; i < l; i++) {
		subs[i].update();
	}
};

Watcher.prototype.update = function update () {
	if (this.lazy) {
		this.dirty = true;
	} else if (this.sync) {
		this.run();
	} else {
		queueWatcher(this);
	}
}

```

执行到 update 内部时你会发现，在最开始将 lazy 设置成了 true，那么这里并不会进入第二分支去执行 watcher.run，也就是说并不会更新 reversedMessage。那问题来了，reversedMessage是怎么更改的呢？在继续讲解之前，我们需要了解的一点是，每个组件实例都对应一个 watcher 实例，在程序初始化是会在 mountComponent 中为 vm 声明一个 watcher，存在全局的 targetStack 中进行维护。

```js
function mountComponent(vm, el, hydrating) {
	updateComponent = function () {
		vm._update(vm._render(), hydrating);
	};

	new Watcher(vm, updateComponent, noop, {
		before: function before () {
			if (vm._isMounted && !vm._isDestroyed) {
				callHook(vm, 'beforeUpdate');
			}
		}
	}, true /* isRenderWatcher */);
}
```

 了解了这个后，再回到上面的 computedGetter，在 watcher.evaluate 执行完之后，紧接着执行下面这一段：

 ```js
if (Dep.target) {
	watcher.depend();
}
 ```

需要注意 watcher.depend 和 dep.depend 执行的逻辑是不同的，前者是将 deps 集合中依次执行 depend，而 dep.depend 是执行单个。此时的 Dep.target 就是 vm watcher。

至于为什么 Dep.target 是 vm watcher，这里可以简单说下：vm watcher 存入之后，targetStack = [vm]。pushTarget(this) 会将 reversedMessage watcher 也存入 targetStaack 中，即 targetStack = [vm, reversedMessage]。get 方法最后会执行 popTarget()，它就是将栈中最后一个弹出，并把 Dep.target 指定给栈中剩下 watcher 的最后一个，也就是 vm，所以 watcher.evaluate() 执行完之后，Dep.target 就是 vm watcher。

```js
Watcher.prototype.depend = function depend () {
	var i = this.deps.length;
	while (i--) {
		this.deps[i].depend();
	}
};
```

reversedMessage.deps 里存放的就是收集它自身的 dep，也就是 message 的监听收集器。

```js
Dep.prototype.depend = function depend () {
	if (Dep.target) {
		Dep.target.addDep(this);
	}
};
```

也就是会将 vm 存入 message 的 dep.subs 中，此时，dep.subs = [reversedMessage, vm]。到这里初始化操作结束了，页面显示为 Computed reversed message: "olleH"。

1秒之后进行 message 的更改操作，重新执行 notify 方法，而 reversedMessage update 依旧不会去执行更新 reversedMessage。。。这仿佛又走的老路子而且还是走不通的。其实不然，因为这次跟上次不同了，这次 dep.subs 中新增了一个 vm 的 watcher，所以执行完 reversedMessage 的 update，接着会执行 vm 的 update 方法。

因为 vm.lazy 和 vm.sync 都是 false，所以会走第三分支，执行 queueWatcher(this)

```js
var queue = [];
var waiting = false;
var has = {};
var flushing = false;

function queueWatcher (watcher) {
	var id = watcher.id;
	if (has[id] == null) {
		has[id] = true;
		if (!flushing) {
			queue.push(watcher);
		} else {
			// if already flushing, splice the watcher based on its id
			// if already past its id, it will be run next immediately.
			var i = queue.length - 1;
			while (i > index && queue[i].id > watcher.id) {
				i--;
			}
			queue.splice(i + 1, 0, watcher);
		}
		// queue the flush
		if (!waiting) {
			waiting = true;

			if (!config.async) {
				flushSchedulerQueue();
				return
			}
			nextTick(flushSchedulerQueue);
		}
	}
}
```

flush 默认为 false，将 vm 添加到 queue 中，执行 nextTick(flushSchedulerQueue)，先来看看 nextTick 方法：

```js
var pending = false;
var callbacks = [];
function nextTick (cb, ctx) {
	var _resolve;
	callbacks.push(function () {
		if (cb) {
			try {
				cb.call(ctx);
			} catch (e) {
				handleError(e, ctx, 'nextTick');
			}
		} else if (_resolve) {
			_resolve(ctx);
		}
	});
	if (!pending) {
		pending = true;
		timerFunc();
	}
}
```

将 flushSchedulerQueue 包装在方法内，存入 callbacks 中，然后去执行 timerFunc()，timerFunc 有如下几种情况：

```js
if (typeof Promise !== 'undefined' && isNative(Promise)) {
	var p = Promise.resolve();
	timerFunc = function () {
		p.then(flushCallbacks);
		if (isIOS) { setTimeout(noop); }
	};
	isUsingMicroTask = true;
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
	isNative(MutationObserver) ||
	MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
	var counter = 1;
	var observer = new MutationObserver(flushCallbacks);
	var textNode = document.createTextNode(String(counter));
	observer.observe(textNode, {
		characterData: true
	});
	timerFunc = function () {
		counter = (counter + 1) % 2;
		textNode.data = String(counter);
	};
	isUsingMicroTask = true;
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
	timerFunc = function () {
		setImmediate(flushCallbacks);
	};
} else {
	timerFunc = function () {
		setTimeout(flushCallbacks, 0);
	};
}
```

就是将 flushCallbacks 作为各种方法的回调函数，而官方的意思是考虑到各平台的差异以及会造成的 bug，按照优先级来说，微任务 > 宏任务，优先使用 promise，最差情况使用 setTimeout。概括来说，就是在数据发生变动时，将所涉及到的 watcher 存入 queue 中，然后使用 nextTick 方法可以让这些 watcher 的更新在下一个时间循环中执行。另外 $nextTick 内部就是执行的 nextTick 方法，因此我们可以使用 $nextTick 在下一个循环中执行方法。

```js
function flushCallbacks () {
	pending = false;
	var copies = callbacks.slice(0);
	callbacks.length = 0;
	for (var i = 0; i < copies.length; i++) {
		copies[i]();
	}
}
```

flushCallbacks 依次执行 callbacks 集合里的方法，当它执行时就会执行 flushSchedulerQueue 方法：

```js

function flushSchedulerQueue () {
	currentFlushTimestamp = getNow();
	flushing = true;
	var watcher, id;

	queue.sort(function (a, b) { return a.id - b.id; });

	for (index = 0; index < queue.length; index++) {
		watcher = queue[index];
		if (watcher.before) {
			watcher.before();
		}
		id = watcher.id;
		has[id] = null;
		watcher.run();
		// in dev build, check and stop circular updates.
		if (has[id] != null) {
			circular[id] = (circular[id] || 0) + 1;
			if (circular[id] > MAX_UPDATE_COUNT) {
				warn(
					'You may have an infinite update loop ' + (
						watcher.user
							? ("in watcher with expression \"" + (watcher.expression) + "\"")
							: "in a component render function."
					),
					watcher.vm
				);
				break
			}
		}
	}

	var activatedQueue = activatedChildren.slice();
	var updatedQueue = queue.slice();

	resetSchedulerState();

	callActivatedHooks(activatedQueue);
	callUpdatedHooks(updatedQueue);

	if (devtools && config.devtools) {
		devtools.emit('flush');
	}
}
```

​简单说下流程：

- 首先要对 queue 里的 watcher 进行排序。原因有三点：
  1. 父组件必须要优先与子组件更新（父组件比子组件先创建）
  2. watcher 分为 user watcher 和 render watcher，user watcher 必须优先与 render watcher（user watcher 比 render watcher 先创建）
  3. 排序后，如果子组件在父组件 watcher 执行过程中被销毁了，就会跳过子组件的 watcher 执行。
- 遍历 queue，如果 watcher.before 存在，优先执行，比如说 beforeUpdate 钩子函数就是在 vm render watcher.before 中执行的；然后执行 watcher.run()
- 执行 activated 和 updated 钩子。
- 开发模式下触发 devtools flush 事件

来看看 watcher.run：

```js
Watcher.prototype.run = function run () {
	if (this.active) {
		var value = this.get();
		if (
			value !== this.value ||
			isObject(value) ||
			this.deep
		) {
			// set new value
			var oldValue = this.value;
			this.value = value;
			if (this.user) {
				var info = "callback for watcher \"" + (this.expression) + "\"";
				invokeWithErrorHandling(this.cb, this.vm, [value, oldValue], this.vm, info);
			} else {
				this.cb.call(this.vm, value, oldValue);
			}
		}
	}
};
```

首先执行 this.get()，触发 this.getter 方法执行，vm.getter 就是 updateComponent，也就是说会重新走 render -> update 过程。而就是在 render 过程里，会重新执行 this.reversedMessage() 更新 reversedMessage 的值。

到这里，将 computed 所涉及到的方法调用整理成一张流程图，如下：

![computed执行流程图](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/computed执行流程图.png)

### 为什么 computed 拥有缓存功能

这个问题其实跟 “如果依赖项不是响应式的，即使改变，computed 为什么也不会重新求值” 这个是一样的。

我在 demo 中增加一个使用 reversedMessage 的代码

```js
<p>Computed reversed message: "{{ reversedMessage }}"</p>
<p>multiple message: "{{ reversedMessage }}"</p>
```

```js
Watcher.prototype.evaluate = function evaluate () {
	this.value = this.get();
	this.dirty = false;
};
```

在 computedGetter 中，只有 this.dirty = true，才会去执行 watcher.evaluate()，也就是去计算 reversedMessage 的值。当第一次使用 reversedMessage 时，当前的 dirty = true，去计算拿到值赋值给 watcher.value，计算完成之后将 dirty 置为 false；第二次使用时就不会去执行 evaluate，拿到的就是第一次计算的值。这样就实现了缓存的功能。

### 如果没有在模板中使用，即使依赖项发生改变，computed 也不会再次执行

修改 demo 如下：

```js
mounted () {
	this.reversedMessage;
	setTimeout(() => {
		this.message = 'cipher'
	}, 1000)
},
computed:{
	reversedMessage: function () {
		console.log('改变 reversedMessage 的值');
		return this.message.split('').reverse().join('')
	}
}
```

这一次在 mounted 钩子中不带() 使用，会触发 reversedMessage 方法执行，但是1秒之后 message 更改了，但是 reversedMessage 却没有执行。

我们看上面 computed 执行流程图，_render() 执行会触发 _s(this.reverseMessage)，进而触发计算，调用方法，但是现在我没有在模板中使用，那么在 render 函数中就不会有 _s(this.reverseMessage)，也就不会有后续的计算操作了，因此 computed 方法不会执行。但是为什么在 mounted 中使用 this.reversedMessage() 会报错，而使用 this.reversedMessage 却可以正常使用？

this.reversedMessage 触发 reversedMessage 的 getter，跟上面提到的执行步骤是一样的，所以会执行 reversedMessage 方法。也就是说 this.reversedMessage() 相当于是 this.reversedMessage()()，this.reversedMessage() 执行完后的值不是方法，所以后面接 () 使用就会报错。

## 最后

computed 的怪异表现其实来自于与 method 的相似性。形式上是 method，但是使用上是 data。

1. 底层上为 computed 添加数据劫持，然后将 computed 代理给了 vm，在当做 data 一样使用时就可以触发 get 去执行这个 computed。
2. 如果不在模板中使用，那么响应式依赖项就算改变了也不会重新求值，是因为重新求值依靠的是页面渲染时触发 computed。
3. 缓存功能其实依赖于 computed 对应 watcher 的 dirty 属性。它的 dirty 默认为 true，只有为 true 才能求值。在进行一次求值后会将 dirty 改为 false，那么下次执行到这里时不会重新求值，而是直接拿上次的值。
