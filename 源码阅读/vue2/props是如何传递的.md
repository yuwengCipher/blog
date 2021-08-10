---
title: props是如何传递的
date: 2021-04-27
categories:
 - Vue
---

## 前言

props 是父组件向子组件传递数据的通道。在子组件中注册一些自定义 prop，当一个值作为 prop attribute 传递给这个 prop 之后，我们在子组件实例中访问这个值就像访问 data 中的值一样。这个设计非常有趣而且简单易用，但你有没有想过底层是如何实现的？我想过而且想知道设计细节，所以这篇就是来寻找答案的！

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

## Vue.component 是什么

vue 初始化程序中会执行 initGlobalAPI(Vue), 用来初始化全局 api 以供后期使用，其中就包括 initAssetRegisters(Vue)。

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

简单来说，就是为 Vue 添加 ASSET_TYPES 类型的属性，属性值是一个方法。那也就是可以知道 Vue.component = function(id, definition) {}，并且它会返回 definition。由于我们调用的时候，满足条件二，所以会对 definition 进行处理：首先是添加 name = 'Child' 属性；因为 this.options._base = Vue，所以接着调用 Vue.extend 处理 definition，即：

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

	Sub.extend = Super.extend;
	Sub.mixin = Super.mixin;
	Sub.use = Super.use;

	ASSET_TYPES.forEach(function (type) {
		Sub[type] = Super[type];
	});
	
	if (name) {
		Sub.options.components[name] = Sub;
	}

	Sub.superOptions = Super.options;
	Sub.extendOptions = extendOptions;
	Sub.sealedOptions = extend({}, Sub.options);

	// cache constructor
	cachedCtors[SuperId] = Sub;
	return Sub
};
```

Super 就是 Vue，将 Sub 的原型指向 Super，并且还将 Vue 本身的属性赋值给 Sub，这样 Sub 就拥有了 Vue 的基本功能，而 Sub 内部执行的就是 Vue._init()，因此 Sub 就可以跟 Vue 一样进行创建组件了。

## 父组件传递属性，子组件接收属性

这些初始化工作做好之后，就开始执行父组件的 new Vue() 流程，执行转译的三个步骤（可以查看“模板怎么变成真实DOM”这一篇）： parse、transform、generate。 其中 transform 会将 AST 转换成如下 render 函数：

```js
(function anonymous(
) {
with(this){return _c('div',{attrs:{"id":"demo"}},[_c('child',{attrs:{"parent-message":message}})],1)}
})
```

子组件的 vnode 创建源于 _c('child',{attrs:{"parent-message":message}}), _c 内部会执行 _createElement 方法，由于符合创建 component 的条件，所以会走 createComponent 方法。

```js
function _createElement (context, tag, data, children, normalizationType) {
	// 省略
	if (typeof tag === 'string') {
		var Ctor;
		ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag);
		if (config.isReservedTag(tag)) {
			// 省略
		} else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
			// component
			vnode = createComponent(Ctor, data, context, children, tag);
		}
	}
}
```

createComponent 参数解释如下：
- Ctor 就是上面说的 Sub，即 function VueComponent() {},
- data: {attrs: parent-message: 'Hello'}。说明一下，这里显示的是 message 的值 Hello，是因为在执行 _c 之前，触发 message get 属性即 'Hello'
- children: undefined
- tag: 'child'。被转换成了小写

```js
// 省略
createComponent(Ctor, data, context, children, tag) {
	// baseCtor 就是 Vue 构造函数
	var baseCtor = context.$options._base;
	var propsData = extractPropsFromVNodeData(data, Ctor, tag);
}

// extractPropsFromVNodeData
var propOptions = Ctor.options.props;
var res = {};
var attrs = data.attrs;
var props = data.props;
if (isDef(attrs) || isDef(props)) {
	for (var key in propOptions) {
		var altKey = hyphenate(key);
		{
			var keyInLowerCase = key.toLowerCase();
			if (
				key !== keyInLowerCase &&
				attrs && hasOwn(attrs, keyInLowerCase)
			) {
				tip(
					"Prop \"" + keyInLowerCase + "\" is passed to component " +
					(formatComponentName(tag || Ctor)) + ", but the declared prop name is" +
					" \"" + key + "\". " +
					"Note that HTML attributes are case-insensitive and camelCased " +
					"props need to use their kebab-case equivalents when using in-DOM " +
					"templates. You should probably use \"" + altKey + "\" instead of \"" + key + "\"."
				);
			}
		}
		checkProp(res, props, key, altKey, true) ||
			checkProp(res, attrs, key, altKey, false);
	}
}
return res
```

extractPropsFromVNodeData 方法中会先获取到 Child 组件中的 props 赋值给 propOptions，即：{parentMessage: {type: null}}；data 中只存在 attrs = {parent-message: "Hello"}，props 是 undefined，但只要其中有一个存在，那么就会去遍历 propOptions：
- altKey 获取的是组件上添加的 attr 名称 "parent-message"，这个在初始化时已经存储起来了，所以这里直接拿就行了（存储过程暂不做解释）
- keyInLowerCase 是将 parentMessage 转成小写的 "parentmessage"
- 最后执行 checkProp(res, attrs, key, altKey, false)。所做的事情就是判断 attrs[altKey] 是否为 true，如果存在就执行 res[key] = attrs[altKey]，即：res['parentMessage'] = attrs['parent-message'] = 'Hello';

propsData 就是最后返回的 res。

获取完 propsData 之后，就要给子组件添加组件管理的钩子函数。

```js
// 先将组件的事件提取出来，因为组件的事件不能按照 Dom 事件进行处理
var listeners = data.on;
// 将原生事件替换组件的事件，因为它可以按照 Dom 事件处理
data.on = data.nativeOn;
installComponentHooks(data);

// installComponentHooks
var hooks = data.hook || (data.hook = {});
for (var i = 0; i < hooksToMerge.length; i++) {
	var key = hooksToMerge[i];
	var existing = hooks[key];
	var toMerge = componentVNodeHooks[key];
	if (existing !== toMerge && !(existing && existing._merged)) {
		hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge;
	}
}
```

全局对象变量 componentVNodeHooks 拥有四个方法：init、prepatch、insert、destroy，hooksToMerge 就是 componentVNodeHooks key 的集合 ['init'、'prepatch'、'insert'、'destroy']。

满足以下条件就可以给 hooks 添加这同 key 方法：
1. hooks[key] !==  componentVNodeHooks[key]
2. hooks[key] 不存在或者 hooks[key]._merged = false

另外如果 hooks[key] 存在，则赋值的是 merge 之后的方法。

最后就是创建 vnode。

```js
var vnode = new VNode(
	("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
	data, undefined, undefined, undefined, context,
	{ Ctor: Ctor, propsData: propsData, listeners: listeners, tag: tag, children: children },
	asyncFactory
);
```

- 在 extend(definition) 内部，将 Sub.cid 赋值为 cid++，因此 Ctor.cid = Sub.cid = 1
- name = 'Child'
- data 拥有了 hook 钩子函数
- propsData = {parentMessage: "Hello"}
- tag = 'child'
- Ctor = function VueComponent() {}

最后一个参整体作为 componentOptions 赋值给 vnode.componentOptions，这是 component 的一个特殊属性。

子组件和父组件 vnode 创建完成后，就开始进入 patch 阶段，创建 Dom，对每一个 vnode 调用 createElm。我们跳过父组件的创建，直接来看 createComponent(vnode, insertedVnodeQueue, parentElm, refElm) 创建 child 组件。parentElm 就是父组件的 dom。特别注意，这个 createComponent 是 createPatchFunction 内部的方法，与上面提到的不是同一个。

```js
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
	var i = vnode.data;
	if (isDef(i)) {
		var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
		if (isDef(i = i.hook) && isDef(i = i.init)) {
			i(vnode, false /* hydrating */);
		}
		// after calling the init hook, if the vnode is a child component
		// it should've created a child instance and mounted it. the child
		// component also has set the placeholder vnode's elm.
		// in that case we can just return the element and be done.
		if (isDef(vnode.componentInstance)) {
			initComponent(vnode, insertedVnodeQueue);
			insert(parentElm, vnode.elm, refElm);
			if (isTrue(isReactivated)) {
				reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
			}
			return true
		}
	}
}
```

首先会调用 hook.init 用来初始化 component，先来看看这个。

```js
init: function init (vnode, hydrating) {
	if (
		vnode.componentInstance &&
		!vnode.componentInstance._isDestroyed &&
		vnode.data.keepAlivez
	) {
		// kept-alive components, treat as a patch
		var mountedNode = vnode; // work around flow
		componentVNodeHooks.prepatch(mountedNode, mountedNode);
	} else {
		var child = vnode.componentInstance = createComponentInstanceForVnode(
			vnode,
			activeInstance
		);
		child.$mount(hydrating ? vnode.elm : undefined, hydrating);
	}
},
```

初始 componentInstance 是不存在的，所以会走第二分支调用 createComponentInstanceForVnode()，将返回的值赋值给 vnode.componentInstance 和变量 child，最后调用 child.$mount() 进行挂载。

其中 createComponentInstanceForVnode 接受的第二个参会作为该组件的 parent，activeInstance 是个全局变量，在 setActiveInstance 方法里会将它赋值为 vm。

```js
function createComponentInstanceForVnode (vnode, parent) {
	var options = {
		_isComponent: true,
		_parentVnode: vnode,
		parent: parent
	};
	// check inline-template render functions
	var inlineTemplate = vnode.data.inlineTemplate;
	if (isDef(inlineTemplate)) {
		options.render = inlineTemplate.render;
		options.staticRenderFns = inlineTemplate.staticRenderFns;
	}
	return new vnode.componentOptions.Ctor(options)
}
```

_isComponent: true 标识它自己是 component;options._parentVnode 就是它自身的 vnode; vnode.componentOptions.Ctor 就是 VueComponent 构造函数，这里执行它，就是执行 _init 方法，创建一个组件实例，就像执行 new Vue() 一样，只不过它们的 options 不同而已。

因为 options._isComponent = true，所以在 _init 方法中不会去 mergeOptions，而是执行 initInternalComponent(vm, options)

```js
// initInternalComponent
var opts = vm.$options = Object.create(vm.constructor.options);
// doing this because it's faster than dynamic enumeration.
var parentVnode = options._parentVnode;
opts.parent = options.parent;
opts._parentVnode = parentVnode;

var vnodeComponentOptions = parentVnode.componentOptions;
opts.propsData = vnodeComponentOptions.propsData;
opts._parentListeners = vnodeComponentOptions.listeners;
opts._renderChildren = vnodeComponentOptions.children;
opts._componentTag = vnodeComponentOptions.tag;

if (options.render) {
	opts.render = options.render;
	opts.staticRenderFns = options.staticRenderFns;
}
```

vm.constructor 指的是 VueComponent，相当于将 vm.$options 的原型指向 VueComponent.options，并且给自身添加多个属性。

## 最后的渲染

组件实例创建完成后，调用 child.$mount() 开始进行 child 组件的 parse、transform、patch 处理过程。其中生成的 render 函数如下所示：

```js
(function anonymous(
) {
with(this){return _c('p',[_v("parentMessage is ï¼š"+_s(parentMessage))])}
})
```

parentMessage 这个属性存在于 $options.propsData 对象中，也会被 observe 系统添加响应式方法，所以这里就会触发 parentMessage.get 方法去拿到它的值 'Hello'，也就实现了 parentMessage 从父组件传递，到子组件接收，再到子组件渲染的这样一个过程。
