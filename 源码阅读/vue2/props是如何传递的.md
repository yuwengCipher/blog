---
title: props是如何传递的
date: 2021-04-27
categories:
 - Vue
---

## 前言

props 是父组件向子组件传递数据的通道。在子组件中注册一些自定义 prop，当一个值作为 prop attribute 传递给这个 prop 之后，在子组件实例中访问这个值就像访问 data 中的值一样。这个设计非常有趣而且简单易用，但你有没有想过底层是如何实现的？我想过而且想知道涉及细节，所以这篇就是来寻找答案的！

按照惯例，还是以官方 dmeo（examples/commits） 为例，稍作改动如下，但需要注意的是在 html 中使用 prop ，传递时需要用短横线分隔命名。

```js
// html
<div id="demo">
	<Child :parent-message="message"></Child>
</div>

// js
Vue.component("Child", {
	props: ['parentMessage'],
	template: '<p>parentMessage is ：{{parentMessage}}</p>'
})

new Vue({
	el: '#demo',
	data: {
		message: 'Hello',
	},
	mounted () {
		setTimeout(() => {
			this.message = 'cipher'
		}, 1000)
	},
})
```

## Vue.component 有什么作用

vue 初始化程序中会执行 initGlobalAPI(Vue), 用来初始化全局 api 以供后期使用，其中就包括 initAssetRegisters(Vue) 这个。

```js
var ASSET_TYPES = [
	'component',
	'directive',
	'filter'
];

function initAssetRegisters (Vue) {
	/**
		* Create asset registration methods.
		*/
	ASSET_TYPES.forEach(function (type) {
		Vue[type] = function (
			id,
			definition
		) {
			if (!definition) {
				return this.options[type + 's'][id]
			} else {
				/* istanbul ignore if */
				if (type === 'component') {
					validateComponentName(id);
				}
				if (type === 'component' && isPlainObject(definition)) {
					definition.name = definition.name || id;
					definition = this.options._base.extend(definition);
				}
				if (type === 'directive' && typeof definition === 'function') {
					definition = { bind: definition, update: definition };
				}
				this.options[type + 's'][id] = definition;
				return definition
			}
		};
	});
}
```

简单来说，就是为 Vue 添加 ASSET_TYPES 类型的属性，属性值是一个方法。那也就是可以知道 Vue.component = function(id, definition) {}; 它会返回 definition。因为我们调用的时候，满足条件二，所以会对 definition 进行处理：首先是添加 name = 'Child' 属性；因为 this.options._base = Vue，所以接着调用 Vue.extend 处理 definition，即：

```js
Vue.extend({
	props: ['parentMessage'],
	data: function () {
		return {
			childMessage: 'hi',
		}
	},
	template: '<p>parentMessage is ：{{parentMessage}}</p>'
})
```

而 Vue.extend 处理如下：

```js
Vue.extend = function (extendOptions) {
	extendOptions = extendOptions || {};
	var Super = this;
	var SuperId = Super.cid;
	var cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
	if (cachedCtors[SuperId]) {
		return cachedCtors[SuperId]
	}

	var name = extendOptions.name || Super.options.name;
	if (name) {
		validateComponentName(name);
	}

	var Sub = function VueComponent (options) {
		this._init(options);
	};
	Sub.prototype = Object.create(Super.prototype);
	Sub.prototype.constructor = Sub;
	Sub.cid = cid++;
	Sub.options = mergeOptions(
		Super.options,
		extendOptions
	);
	Sub['super'] = Super;

	if (Sub.options.props) {
		initProps(Sub);
	}
	if (Sub.options.computed) {
		initComputed(Sub);
	}

	// allow further extension/mixin/plugin usage
	Sub.extend = Super.extend;
	Sub.mixin = Super.mixin;
	Sub.use = Super.use;

	// create asset registers, so extended classes
	// can have their private assets too.
	ASSET_TYPES.forEach(function (type) {
		Sub[type] = Super[type];
	});
	// enable recursive self-lookup
	if (name) {
		Sub.options.components[name] = Sub;
	}

	// keep a reference to the super options at extension time.
	// later at instantiation we can check if Super's options have
	// been updated.
	Sub.superOptions = Super.options;
	Sub.extendOptions = extendOptions;
	Sub.sealedOptions = extend({}, Sub.options);

	// cache constructor
	cachedCtors[SuperId] = Sub;
	return Sub
};
```

Super 就是 Vue，将 Sub 的原型指向 Super，并且还将 Vue 本身的属性赋值给 Sub，这样 Sub 就拥有了 Vue 的基本功能，而 Sub 内部执行的就是 Vue._init()，因此 Sub 就可以跟 Vue 一样进行创建组件了。

这些初始化工作做好之后，就开始执行父组件的 new Vue() 流程，当进入 patch 流程时（如果不清楚 patch 流程，可以查看“模板怎么变成真实DOM”这一篇），当执行 createElm 时，会先执行 createComponent 方法去验证是否是 component ，执行创建 component 工作：

```js
// 省略
createComponent(vnode, insertedVnodeQueue, parentElm, refElm)
```
