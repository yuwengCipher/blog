---
title: 彻底弄懂 VueRouter 工作原理
date: 2021-07-25
categories:
 - Vue
tags:
 - VueRouter
---

## 前言

作为 Vue 生态的一部分，VueRouter 其实跟 Vuex 一样，是作为一个插件存在的。有了它，我们就拥有了前端路由，就可以使用 Vue 创建 SPA，然后 SPA 带给我们的是不刷新页面就可以更新内容的良好体验，可以说 VueRouter 给 Vue 添上了一对翅膀。这一篇就是通过例子结合源码的方式做的一些总结。

## 实现路由的基础

SPA 应用可以理解成由一个中枢管理多个 route——>UI 映射集合的工作的程序。这里就涉及到两个因素，一个是 route 的变化，一个是 UI 的更新。如何监控 route 改变以及更新 UI 就是重点。

我们在实例化 VueRouter 时，会配置路由模式，官方提供 hash 和 history 两种。相对应的，浏览器也有它的接口来处理这两种情况。先来了解一下浏览器会如何处理 route 的变化。

### url 历史记录管理

按我的理解，浏览器设有两个栈来管理 url 历史记录，一个是当前栈，一个是备用栈。当点击后退按钮，当前栈顶的 url 会被弹出栈，此时不会被销毁，而是进入备用栈中；当点击前进按钮就会从备用栈取出第一个 url 放入当前栈显示。但是每当新的 url 键入地址栏中并显示页面时，这个 url 就会存入当前栈，并且备用栈里的 url 都会被销毁。如果备用栈中没有记录，前进按钮是置灰不能点击的。![历史记录栈](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/历史记录栈.png)

### hash

在 hash 模式下，地址栏的 url 会显示为类似 <http://xxxxx/#/foo> 这种，#/foo 这部分称为 url 的片段标识符，也就是 hash，它发生变化时浏览器不会刷新。通过浏览器前进、后退按钮会改变 hash 值，使用 window.onhashchange 可以监听这个改变动作。

### history

window.history 提供了 pushState、replaceState、go、back、forward 方法去改变 url；使用 popstate 可以监听 url 变化。其中 popstate 无法监听到 pushState 和 repalceState。

## 路由实例挂载

我们在应用中使用 router 时，通常会先按下面步骤执行：

- Vue.use(VueRouter)
- const router = new VueRouter({})
- new Vue({router})

执行完这三个步骤，VueRouter 就融入到 Vue 中，然后我们就可以按照既定的路径去浏览应用。那么接下来就看看每一步所做的工作。

### Vue.use(VueRouter)

Vue.use 是 Vue 加载插件的通用方法。

```js
Vue.use = function (plugin) {
	var installedPlugins = (this._installedPlugins || (this._installedPlugins = []));
	if (installedPlugins.indexOf(plugin) > -1) {
		return this
	}

	var args = toArray(arguments, 1);
	args.unshift(this);
	if (typeof plugin.install === 'function') {
		plugin.install.apply(plugin, args);
	} else if (typeof plugin === 'function') {
		plugin.apply(null, args);
	}
	installedPlugins.push(plugin);
	return this
};
```

Vue 内部会使用 _installedPlugins 来维护已经加载过的插件，多次 use 同一个插件，它会从这个栈中找到这个插件并返回。

对于初次加载的插件，则会优先去插件内部去调用它的安装方法即 install，如果插件本身是一个函数，那么就调用它本身。而传入的参数 args 是包括 vue 和 插件的数组。

调用完成后，说明这个插件已经加载完了，它就会被存进栈中维护起来。

VueRouter 的 install 方法如下：

```js
var _Vue;
function install (Vue) {
	if (install.installed && _Vue === Vue) { return }
	install.installed = true;

	_Vue = Vue;

	var isDef = function (v) { return v !== undefined; };

	var registerInstance = function (vm, callVal) {
		var i = vm.$options._parentVnode;
		if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
			i(vm, callVal);
		}
	};

	Vue.mixin({
		beforeCreate: function beforeCreate () {
			if (isDef(this.$options.router)) {
				this._routerRoot = this;
				this._router = this.$options.router;
				this._router.init(this);
				Vue.util.defineReactive(this, '_route', this._router.history.current);
			} else {
				this._routerRoot = (this.$parent && this.$parent._routerRoot) || this;
			}
			registerInstance(this, this);
		},
		destroyed: function destroyed () {
			registerInstance(this);
		}
	});

	Object.defineProperty(Vue.prototype, '$router', {
		get: function get () { return this._routerRoot._router }
	});

	Object.defineProperty(Vue.prototype, '$route', {
		get: function get () { return this._routerRoot._route }
	});

	Vue.component('RouterView', View);
	Vue.component('RouterLink', Link);

	var strats = Vue.config.optionMergeStrategies;
	strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created;
}
```

在 install 方法里会在加载完成时为它自身添加 installed 属性，并且对 VueRouter 的 _Vue 赋值，这两个属性标识它是否已经被 Vue 加载过。

接着是使用 Vue.mixin 向 Vue 组件混入 beforeCreate、destroyed 这两个生命周期钩子，意思就是在组件创建之前注册路由实例，组件销毁之后销毁路由实例。至于怎么注册以及销毁我们后面再说。

然后就是为 Vue.prototype 添加 $router 和 $route 属性，分别对应全局路由实例和当前路由信息对象。另外就是注册 RouterView 和 RouterLink 组件，之后我们就可以这样使用： <router-view></router-view> 和 <router-link></router-link>。

最后是将路由生命周期钩子函数的合并策略统一成 Vue 组件声明周期钩子合并策略。

### new VueRouter({})

这一步是创建路由实例。

```js
var VueRouter = function VueRouter (options) {
	if (options === void 0) options = {};

	// app 即 Vue 
	this.app = null;
	this.apps = [];
	this.options = options;
	// 这三个会存放在路由钩子执行的回调，
	// 分别对应 beforeEach、beforeResolve、afterEach
	this.beforeHooks = [];
	this.resolveHooks = [];
	this.afterHooks = [];
	// 根据 routes 去创建针对 routes 的操作方法，比如匹配，添加等。
	this.matcher = createMatcher(options.routes || [], this);

	var mode = options.mode || 'hash';
	this.fallback =
		mode === 'history' && !supportsPushState && options.fallback !== false;
	if (this.fallback) {
		mode = 'hash';
	}
	if (!inBrowser) {
		mode = 'abstract';
	}
	this.mode = mode;

	switch (mode) {
		case 'history':
			this.history = new HTML5History(this, options.base);
			break
		case 'hash':
			this.history = new HashHistory(this, options.base, this.fallback);
			break
		case 'abstract':
			this.history = new AbstractHistory(this, options.base);
			break
		default:
			{
				assert(false, ("invalid mode: " + mode));
			}
	}
};
```

通过构造函数可以看出，路由模式默认为 hash。并且根据不同的模式，会初始化不同的 History 实例。HTML5History 和 HashHistory 大致结构一样，内部都是继承自 History，只是各自拥有方法的处理细节不同，下面以 HashHistory 为例简单讲讲。

下面就是 History 的基本组成构造。

```js
var History = function History (router, base) {
	// 全局路由对象信息
	this.router = router;
	this.base = normalizeBase(base);
	// 当前的路由对象，默认为不指向任何地方
	this.current = START;
	this.pending = null;
	this.ready = false;
	this.readyCbs = [];
	this.readyErrorCbs = [];
	this.errorCbs = [];
	this.listeners = [];
};
// 添加回调
History.prototype.listen = function listen (cb) {};
History.prototype.onReady = function onReady (cb, errorCb) {};
History.prototype.onError = function onError (errorCb) {};
// 路由跳转
History.prototype.transitionTo = function transitionTo () {};
// transitionTo 内部会调用的方法
History.prototype.confirmTransition = function confirmTransition (route, onComplete, onAbort) {};
// 更新路由信息
History.prototype.updateRoute = function updateRoute (route) {};
// 路由发生改变执行的回调
History.prototype.setupListeners = function setupListeners () {};
// 组件销毁时去重置 history
History.prototype.teardown = function teardown () {};
```

现在就来看看 HashHistory。它其实跟 History 组成类似，就是构造函数 + 原型方法。

```js
function HashHistory (router, base, fallback) {
	History.call(this, router, base);
	// check history fallback deeplinking
	if (fallback && checkFallback(this.base)) {
		return
	}
	ensureSlash();
}

if (History) HashHistory.__proto__ = History;
HashHistory.prototype = Object.create(History && History.prototype);
HashHistory.prototype.constructor = HashHistory;
```

首先是对 History 常规的继承处理。其中的 ensureSlash 是为了确保 "#" 后面第一个字符是 "/"。
而重点是它的原型方法，这里只罗列两个常用的。

```js
// 路由对象发生改变时会执行的方法
HashHistory.prototype.setupListeners = function setupListeners () {}
// 我们执行 this.$router.push() 实际上执行的方法
HashHistory.prototype.push = function push (location, onComplete, onAbort) {}
// 我们执行 this.$router.replace() 实际上执行的方法
HashHistory.prototype.replace = function replace (location, onComplete, onAbort) {}
```

创建完成的 history 实例就包括了 History 函数的属性和方法以及 HashHistory 的方法。

### new Vue({router})

到这里开始创建 Vue 实例、挂载组件。传入的 router 实例对象被存放在 vue.$options.router 中。上面有提到过一点：VueRouter 的 install 方法向 Vue 混入了 beforeCreate、destroyed 钩子函数，而组件在创建时都会去执行 beforeCreate 这个钩子。

```js
beforeCreate: function beforeCreate () {
	if (isDef(this.$options.router)) {
		this._routerRoot = this;
		this._router = this.$options.router;
		this._router.init(this);
		Vue.util.defineReactive(this, '_route', this._router.history.current);
	} else {
		this._routerRoot = (this.$parent && this.$parent._routerRoot) || this;
	}
	registerInstance(this, this);
}
```

我们知道父子组件中 beforeCreate 的执行顺序是 父beforeCreate——>子beforeCreate，因此会首先执行根组件的 beforeCreate。此时 this.$options.router 是存在的，执行 VueRouter 的 init 方法去初始化。

其中还将 _routerRoot 指定为根组件自身，这个非常关键，子组件不存在 this.$options.router，那么就会进入第二个分支，优先将父组件的 _routerRoot 赋值给自身的 _routerRoot，然后子组件也可以像根组件一样通过 this.$router 获取 router 对象了。也就是通过这种方式将 router 对象层层注入到全局组件，所有组件使用同一个 router。

另外就是通过 defineReactive 为所有组件添加 _route 属性，指向当前路由对象，结合 install 方法中对 Vue.prototype 的数据劫持，所有组件也就能通过 this.$route 获取当前路由对象。

接下来继续看下 router 的初始化方法 init，简化之后如下：

```js
var history = this.history;

if (history instanceof HTML5History || history instanceof HashHistory) {
	// 用于控制页面展示时 scroll 的位置
	var handleInitialScroll = function (routeOrError) {
		var from = history.current;
		var expectScroll = this$1$1.options.scrollBehavior;
		var supportsScroll = supportsPushState && expectScroll;

		if (supportsScroll && 'fullPath' in routeOrError) {
			handleScroll(this$1$1, routeOrError, from, false);
		}
	};
	// 路由更改执行的方法
	var setupListeners = function (routeOrError) {
		history.setupListeners();
		handleInitialScroll(routeOrError);
	};
	history.transitionTo(
		history.getCurrentLocation(),
		setupListeners,
		setupListeners
	);
}

history.listen(function (route) {
	this$1$1.apps.forEach(function (app) {
		app._route = route;
	});
});
```

这里实际上只有两个操作：一个是执行 history.transitionTo() 去更改路由；一个是执行 history.listen 添加一个回调，在 history 路由更改完成后去更新 Vue 的 _route。

transitionTo 接收三个参数，分别是需要跳转的路由信息、操作成功执行的回调和操作失败执行的回调。这里的两个回调是同一个函数，它的内部会执行 history.setupListeners()。

```js
HashHistory.prototype.setupListeners = function setupListeners () {
	var this$1$1 = this;

	// 省略

	var handleRoutingEvent = function () {
		var current = this$1$1.current;
		if (!ensureSlash()) {
			return
		}
		this$1$1.transitionTo(getHash(), function (route) {
			if (supportsScroll) {
				handleScroll(this$1$1.router, route, current, true);
			}
			if (!supportsPushState) {
				replaceHash(route.fullPath);
			}
		});
	};
	var eventType = supportsPushState ? 'popstate' : 'hashchange';
	window.addEventListener(
		eventType,
		handleRoutingEvent
	);
	this.listeners.push(function () {
		window.removeEventListener(eventType, handleRoutingEvent);
	});
};
```

虽然路由模式是 hash，但还是优先使用 popstate 事件去监听路由的变化。回调函数内部会执行 transition 方法。在文章开头我们说过 popstate 可以监听浏览器的 back、forward 事件，也就是说当我们点击浏览器的前进、后退按钮，会触发监听的事件去执行 transition 更改路由。

## 路由变化方式

SPA 页面的切换其实主要由三个步骤组成：
- 地址栏 url 变化
- history 路由信息更换
- component 切换

执行完这三个步骤，页面的显示就可以从一个组件切换到另一个组件。

这一小节主要来说说前两个步骤在 Vue 中的表现形式。

上面讲到了通过 popstate 去监听浏览器的前进后退事件，前进后退是改变路由的一种方式， 除此之外，Vue 中路由变化的方式主要有如下几种：

> this.$router.push()
> this.$router.replace()
> this.$router.go();
> this.$router.back();
> this.$router.forward();
> router-link

上面方法对应如下：

```js
VueRouter.prototype.push = function push (location, onComplete, onAbort) {
	this.history.push(location, onComplete, onAbort);
}

VueRouter.prototype.replace = function push (location, onComplete, onAbort) {
	this.history.replace(location, onComplete, onAbort);
}

VueRouter.prototype.go = function go (n) {
	this.history.go(n);
};

VueRouter.prototype.back = function back () {
	this.go(-1);
};

VueRouter.prototype.forward = function forward () {
	this.go(1);
};
```

可以看到内部其实都是执行 history 的方法。

其中 push 和 replace 内部会去执行 history.transitionTo，以 push 为例：

```js
HashHistory.prototype.push = function push (location, onComplete, onAbort) {
	// 省略
	this.transitionTo(
		location,
		function (route) {
			pushHash(route.fullPath);
			//省略
			onComplete && onComplete(route);
		},
		onAbort
	);
};

function pushHash (path) {
	if (supportsPushState) {
		pushState(getUrl(path));
	} else {
		window.location.hash = path;
	}
}
```

transitionTo 更改路由信息，在成功回调内会去执行 pushHash 方法去更换地址栏 url。

而 go 实际上是会去调用 window.history.go 方法改变地址栏 url，而且这个方法会触发上面监听的 popstate 方法去调用 history.transitionTo 改变路由信息。back、forward 同理。

```js
HashHistory.prototype.go = function go (n) {
	window.history.go(n);
};
```

最后就来说说 RouterLink 组件了。它的用法如下：

```js
<router-link to="/foo">/foo</router-link>
```

当点击这个组件时，就会执行类似于 $router.push 去切换路由。它是一个内部组件，拥有自身的 render 函数，简化后如下：

```js
var Link = {
	props: {
		to: {
			type: toTypes,
			required: true
		},
		tag: {
			type: String,
			default: 'a'
		},
		replace: Boolean,
	},
	render: function() {
		var handler = function (e) {
			if (guardEvent(e)) {
				if (this$1$1.replace) {
					router.replace(location, noop);
				} else {
					router.push(location, noop);
				}
			}
		};

		var on = { click: guardEvent };
		if (Array.isArray(this.event)) {
			this.event.forEach(function (e) {
				on[e] = handler;
			});
		} else {
			on[this.event] = handler;
		}
		return h(this.tag, data, this.$slots.default)
	}
}
```

这个组件默认会被渲染成 a 标签，还会给自身添加上点击事件，触发回调事件为 handler。如果组件上注明了 replace 属性，那么就调用 router.replace()，否则执行 router.push()。也就是说 RouterLink 的执行原理与上面说到的一样。

## RouterView

跟路由相关的内置指令还有 RouterView，但跟 RouterLink 不一样的是它不会渲染成某个标签，类似于 fragment，而是渲染成匹配到的组件，那何为匹配到的组件呢？路由是可以嵌套的，那么就会按照嵌套的层级进行分层，每一层的组件会被一个 RouterView 所维护渲染，默认只有一个根 RouterView。具体可查看[Vue Router](https://router.vuejs.org/zh/)。下面具体看看 RouterView 是如何渲染匹配组件吧。

上面提到的路由实例挂载步骤第二步中，对根组件混入了 beforeCreate 钩子，其中在执行 router 的初始化方法时会默认调用一次 history.transitionTo 方法，而在该方法内部会确定当前的 route。

第三步就是创建 Vue 实例，然后进行挂载。此过程在 [模板怎么变成真实DOM](https://djacipher.cn/2021/05/02/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/vue2/%E6%A8%A1%E6%9D%BF%E6%80%8E%E4%B9%88%E5%8F%98%E6%88%90%E7%9C%9F%E5%AE%9E%20DOM/) 这篇文章中已经讲过，不在重复。我们在这里只需关注涉及到的 render 方法（此 render 方法来自于例子 basic 处理后）

```js
(function anonymous(
) {
with(this){return _c('div',{attrs:{"id":"app"}},[_c('h1',[_v("Basic")]),_v(" "),_c('ul',[_c('li',[_c('router-link',{attrs:{"to":"/"}},[_v("/")])],1),_v(" "),_c('li',[_c('router-link',{attrs:{"to":"/foo"}},[_v("/foo")])],1),_v(" "),_c('li',[_c('router-link',{attrs:{"to":"/bar"}},[_v("/bar")])],1)]),_v(" "),_c('button',{attrs:{"id":"navigate-btn"},on:{"click":navigateAndIncrement}},[_v("On Success")]),_v(" "),_c('pre',{attrs:{"id":"counter"}},[_v(_s(n))]),_v(" "),_c('pre',{attrs:{"id":"query-t"}},[_v(_s($route.query.t))]),_v(" "),_c('pre',{attrs:{"id":"hash"}},[_v(_s($route.hash))]),_v(" "),_c('router-view',{staticClass:"view"})],1)}
})
```

render 方法跟路由相关的主要有三处：
- _s($route.query.t)
- _s($route.hash)
- _c('router-view',{staticClass:"view"})

其中前两块儿有一个共同的作用，那就是帮助 _route 完成依赖收集。为什么这样说呢？

首先 VueRouter.install 做了两件事，一个是为 Vue.prototype 添加 $route 属性，并且为其添加 get 方法

```js
Object.defineProperty(Vue.prototype, '$route', {
	get: function get () { return this._routerRoot._route }
});
```

另一个是在 beforeCreate 中通过 Vue.util.defineReactive(this, '_route', this._router.history.current) 为 _route 添加数据劫持。
而使用 Vue.$route 实际上是使用 this._routerRoot._route，那么也就会触发 _route 的 get 方法。

知道这样一个关系之后，我们就可以知道当执行 _s($route.query.t) 时会触发 _route 的 get 方法，此时的 dep.target 就是 render watcher，那么 render watcher 就会被收集到它的依赖中，每当 _route 改变的时候，render watcher 的表达式就会执行，也就会重新执行 render 方法。

那 _c('router-view',{staticClass:"view"}) 做了些什么呢？

_c 实际是执行 createComponent，对于 router-view 会执行 createFunctionalComponent 创建组件，原因就在于 router-view 组件指明了自身为 functional 为 true。

```js
var View = {
	name: 'RouterView',
	functional: true,
	props: {
		name: {
			type: String,
			default: 'default'
		}
	},
	render: function render (_, ref) {}
```

在 createFunctionalComponent 内部会执行 var vnode = options.render.call(null, renderContext._c, renderContext) 这一段代码获取 vnode，那也就是执行 RouterView 自带的 render 方法，简化如下所示：

```js
render: function render (_, ref) {
  var parent = ref.parent;
	var h = parent.$createElement;
	var name = props.name;
	var route = parent.$route;
	var cache = parent._routerViewCache || (parent._routerViewCache = {});

  var depth = 0;
	var matched = route.matched[depth];
	var component = matched && matched.components[name];

	if (!matched || !component) {
		cache[name] = null;
		return h()
	}

	return h(component, data, children)
}
```

ref 就是 RouterView 组件对象，这个例子中的 RouterView 处于根组件内，因此 ref.parent 就是根组件，并且路由层级为0，通过 route.matched[depth] 拿到该层级当前的路由信息，然后通过路由名称去拿到路由对应的组件，这里的名称默认为 default。如果成功拿到了组件，就使用父组件的 createElement 渲染该组件，否则渲染空组件。

完成初次渲染之后，_route 也完成了 render watcher 的收集，之后不论是以哪一种方式去改变路由，都会触发 render，进而重新显示对应的组件。

![RouterView](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/RouterView.png)

## 导航守卫

Vue 提供的生命周期钩子让我们在开发程序时可以轻松自如的在不同时机处理不同的事情，同样的，VueRouter 也提供了各种钩子，这里称为导航守卫。依据声明的位置不同，可以分为全局级、路由级、组件级三种，它的主要作用是可以让我们去取消一次路由跳转或重定向路由。可以查看 [路由导航](https://router.vuejs.org/guide/advanced/navigation-guards.html)

### 全局级

这一层接在 router.js 中声明，分为 beforeEach、beforeResolve、afterEach，如：

```js
const router = new VueRouter({ ... })

router.beforeEach((to, from, next) => {
  if (to.query.delay) {
    setTimeout(() => {
      next()
    }, Number(to.query.delay))
  } else {
    next()
  }
})
```

### 路由级

路由级只有一个 beforeEnter，我们可以在每一个路由对象中去添加，如：

```js
const router = new VueRouter({
  routes: [
    {
      path: '/foo',
      component: Foo,
      beforeEnter: (to, from, next) => {}
    }
  ]
})
```

### 组件级

组件级有三个守卫，即：beforeRouteEnter、beforeRouteUpdate、beforeRouteLeave。在组件中声明如下：

```js
const Foo = {
  template: `...`,
  beforeRouteEnter(to, from, next) {},
  beforeRouteUpdate(to, from, next) {},
  beforeRouteLeave(to, from, next) {}
}
```

既然提供了这些守卫，那它们的执行顺序是什么呢？官网也提供了守卫的执行流，表现如下：
1. Navigation triggered
2. beforeRouteLeave in deactived components
3. beforeEach
4. beforeRouteUpdate in reused components
5. beforeEnter
6. resolve async component
7. beforeRouteEnter in actived components
8. beforeReolve
9. Navigation confirmed
10. afterEach
11. DOM update triggered

那是如何实现这样一种执行流顺序呢？应该记得我们上面说过，路由跳转底层执行的是 transitionTo 方法，

```js
History.prototype.transitionTo = function transitionTo (){
	// 忽略
	this.confirmTransition()
}
```

它内部实际执行的是 confirmTransition，即：

```js
History.prototype.confirmTransition = function confirmTransition (route, onComplete, onAbort) {
	// 忽略
	var queue = [].concat(
		// in-component leave guards
		extractLeaveGuards(deactivated),
		// global before hooks
		this.router.beforeHooks,
		// in-component update hooks
		extractUpdateHooks(updated),
		// in-config enter guards
		activated.map(function (m) { return m.beforeEnter; }),
		// async components
		resolveAsyncComponents(activated)
	);

	runQueue(queue, iterator, function () {
		// wait until async components are resolved before
		// extracting in-component enter guards
		var enterGuards = extractEnterGuards(activated);
		var queue = enterGuards.concat(this$1$1.router.resolveHooks);
		runQueue(queue, iterator, function () {
			if (this$1$1.pending !== route) {
				return abort(createNavigationCancelledError(current, route))
			}
			this$1$1.pending = null;
			onComplete(route);
			if (this$1$1.router.app) {
				this$1$1.router.app.$nextTick(function () {
					handleRouteEntered(route);
				});
			}
		});
	});
}
```

可以看到，使用一个队列顺序存储需要执行的守卫方法及其他处理，然后使用 runQueue 去执行，就可以按顺序执行里面的处理。其中队列最后一个处理是 reolve components，而 beforeRouteEnter 是在 runQueue 回调方法内执行 ，这也就确保它的执行是在 resolve components 之后。

## 最后

- 浏览器提供的原生接口是实现前端路由的基石
- 将 Router 放入 Vue 中总的来说需要三个步骤，前两个步骤就是初始化 Router 实例及 History 对象，后一个步骤是正常的实例化 Vue 对象实现组件的挂载
- 由于前两个步骤做了前置动作，组件在渲染时会执行混入的钩子函数，初始化路由对象并完成路由的初次 transitionTo。然后就是组件正常的渲染，在执行 render 方法时完成对 render watcher 的依赖收集，这样之后，每当 route 改变，render watcher 的 expression 就会执行去更新页面，也就可以渲染显示新路由对应的组件，简单来说就是 route 改变 ——> render ——> 渲染 router-view。
