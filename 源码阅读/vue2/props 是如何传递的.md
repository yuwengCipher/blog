---
title: props 是如何传递的
date: 2021-05-23
categories:
 - Vue
tags:
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

我们使用 Vue.component() 注册一个组件，然后就可以使用这个组件。那这个方法是如何注册组件的呢？我们知道 vue 初始化程序时会执行 initGlobalAPI(Vue) 初始化全局 api 以供后期使用，其中就包括 initAssetRegisters(Vue)。

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

Super 就是 Vue，将 Sub.prototype 的原型指向 Super.prototype，并且还将 Vue 本身的属性赋值给 Sub，这样 Sub 就拥有了 Vue 的基本功能，而 Sub 内部执行的就是 Vue._init()，因此 Sub 就可以跟 Vue 一样进行创建组件了。

## 父组件传递属性，子组件接收属性

这些初始化工作做好之后，就开始执行父组件的 new Vue() 流程，执行转译的三个步骤： parse、transform、generate。转译步骤可以查看 [模板怎么变成真实DOM](https://djacipher.cn/2021/05/02/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/vue2/%E6%A8%A1%E6%9D%BF%E6%80%8E%E4%B9%88%E5%8F%98%E6%88%90%E7%9C%9F%E5%AE%9E%20DOM/) 。其中 transform 会将 AST 转换成如下 render 函数：

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
- data: {attrs: parent-message: 'Hello'}。说明一下，这里显示的是 message 的值 Hello，是因为在执行 _c 之前，触发 message get 属性就能拿到它的值 'Hello'
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

extractPropsFromVNodeData 方法中会先获取到 Child 组件中的 props 赋值给 propOptions，即：{parentMessage: {type: null}}；data 中只存在 attrs = {parent-message: "Hello"}，props 是 undefined，但只要其中有一个存在就会去遍历 propOptions：
- altKey 就是通过正则将 parentMessage 转换为 "parent-message"
- keyInLowerCase 是将 parentMessage 转成小写的 "parentmessage"
- 最后执行 checkProp(res, attrs, key, altKey, false)。所做的事情就是执行 res[key] = attrs[altKey]，即：res['parentMessage'] = attrs['parent-message'] = 'Hello';

这里要讲一下 checkProp

```js
function checkProp (res, hash, key, altKey, preserve) {
	if (isDef(hash)) {
		if (hasOwn(hash, key)) {
			res[key] = hash[key];
			if (!preserve) {
				delete hash[key];
			}
			return true
		} else if (hasOwn(hash, altKey)) {
			res[key] = hash[altKey];
			if (!preserve) {
				delete hash[altKey];
			}
			return true
		}
	}
	return false
}
```

如果 attrs 或者 props 没有值，那么就会直接返回 false；如果存在，就有可能出现两种情况：

第一种是使用 :parentMessage="message" 这种形式绑定属性，那么就会匹配第一个分支，执行 res['parentMessage'] = attrs['parentMessage'];第二种就是 demo 的这种形式，执行 res['parentMessage'] = attrs['parent-message']。

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
		// 省略
	}
}
```

调用 hook.init 初始化 component。

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

其中 createComponentInstanceForVnode 接收的第二个参会作为该组件的 parent，activeInstance 是个全局变量，在 setActiveInstance 方法里会将它赋值为 vm。

```js
function createComponentInstanceForVnode (vnode, parent) {
	var options = {
		_isComponent: true,
		_parentVnode: vnode,
		parent: parent
	};
	var inlineTemplate = vnode.data.inlineTemplate;
	if (isDef(inlineTemplate)) {
		options.render = inlineTemplate.render;
		options.staticRenderFns = inlineTemplate.staticRenderFns;
	}
	return new vnode.componentOptions.Ctor(options)
}
```

_isComponent: true 标识它自己是 component; options._parentVnode 就是它自身的 vnode;  vnode.componentOptions.Ctor 就是 VueComponent 构造函数，这里执行它，就是执行 _init 方法，创建一个组件实例，就像执行 new Vue() 一样，只是 options 不同而已。

在 initState 中会去执行 initProps(vm, opts.props) 处理 props：

```js
// propsOptions = opts.props = {parentMessage: {type: null}}
function initProps (vm, propsOptions) {
	// propsdata = {parentMessage: 'Hello'}
	var propsData = vm.$options.propsData || {};
	var props = vm._props = {};
	var loop = function (key) {
		// value 即：'Hello'
		var value = validateProp(key, propsOptions, propsData, vm);
		{
			defineReactive(props, key, value, function () {
				if (!isRoot && !isUpdatingChildComponent) {
					warn(
						"Avoid mutating a prop directly since the value will be " +
						"overwritten whenever the parent component re-renders. " +
						"Instead, use a data or computed property based on the prop's " +
						"value. Prop being mutated: \"" + key + "\"",
						vm
					);
				}
			});
		}
	};
	for (var key in propsOptions) loop(key);
}
```

它所做的事有两点：
1. 声明一个变量 props，将它与 vm._props 都指向 {}，如果 props 发生改变，vm._props 也会跟着改变
2. 为 props.parentMessage 添加数据劫持并且赋值为 'Hello'。

通过断点调试，你会发现执行完这一步后，vm.prototype 添加上了 parentMessage 属性，值也是 'Hello'。这是为什么呢？在开始去寻找答案之前，我们可以猜测是 vm._props 与 vm.prototype 自身做了某种联系才会有这样的同步效果。因为 vm 的构造函数是 VueComponent，我们找到最上面的 extend 方法，里面有一个这样的处理：

```js
if (Sub.options.props) {
	initProps(Sub);
}
```

> Sub.options.props = {parentMessage: {type: null}}

进入 initProps(Sub) 处理流程：

```js
function initProps (Comp) {
	var props = Comp.options.props;
	for (var key in props) {
		proxy(Comp.prototype, "_props", key);
	}
}

function proxy (target, sourceKey, key) {
	sharedPropertyDefinition.get = function proxyGetter () {
		return this[sourceKey][key]
	};
	sharedPropertyDefinition.set = function proxySetter (val) {
		this[sourceKey][key] = val;
	};
	Object.defineProperty(target, key, sharedPropertyDefinition);
}
```

由上可知，proxy 的作用就是为 target[key] 添加响应式属性 get/set。这里就是为 VueComponent.prototype[parentMessage] 添加属性，get 方法获取的是 vm._props.parentMessage。这里相当于使用 props 需要通过两层代理：

第一层：vm.prototype.parentMessage ——> vm.prototype.parentMessage.get() ——> return vm._props.parentMessage
第二层：vm._props.parentMessage ——> vm._props.parentMessage.get() ——> return 'Hello'

串起来就是 vm.prototype.parentMessage ——> vm._props.parentMessage。

## 最后的渲染

组件实例创建完成后，调用 child.$mount() 开始进行 child 组件的 parse、transform、patch 处理过程。其中生成的 render 函数如下所示：

```js
(function anonymous(
) {
with(this){return _c('p',[_v("parentMessage is ï¼š"+_s(parentMessage))])}
})
```

当使用 parentMessage 这个属性时，会触发 vm.prototype.parentMessage 的 get 方法，拿到 'Hello'，那么 'Hello' 就可以被渲染到页面中了。也就实现了 parentMessage 从父组件传递，到子组件接收，再到子组件渲染的这样一个过程。用一张图来表示一下整个流程：

![props传递流程图](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/props传递流程图.png)

## 为什么 props 的值会跟着父组件改变

message 会在1秒之后变成 cipher，而 parentMessage is：Hello 也会显示为 parentMessage is：cipher，为什么 props 的值会跟着父组件改变？

在 [computed的[怪异]表现](https://djacipher.cn/2021/05/12/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/vue2/computed%20%E7%9A%84%5B%E6%80%AA%E5%BC%82%5D%E8%A1%A8%E7%8E%B0/) 中已经讲过，message 的改变会触发父组件的重新渲染，那么就会将我们上面说到的流程重新走一遍，而这次传进来的 message 是 cipher，所以也就会显示 cipher。

## 子组件如何更改 prop

在 initProps 中为 prop 绑定数据劫持时，会传入一个 customSetter 方法，当我们试图改变 props 时，就会执行这个方法，提示不能在直接在子组件更改属性，需要父组件去更改，然后子组件来更新。官方给出的方案是 .sync 修饰符。我们将 demo 改成如下所示：

```js
// html
<Child :parent-message.sync="message"></Child>

// js
// 将 parentMessage 的更改挪到 child 中
mounted () {
	setTimeout(() => {
		this.$emit('update:parentMessage', 'change in child')
	}, 1000)
},
```

这样修改之后，1秒之后，页面就从 parentMessage is：Hello 显示为 parentMessage is：change in child。实现了在子组件去更改父组件的 prop。

想知道原因，我们就得从源头下手，那就是父组件的 parse 阶段，在这个阶段会将 template 转换成 AST，会发现 child 的 AST.attrsList 属性如下：

```js
attrsList = [
	{
		end: 57
		name: ":parent-message.sync"
		start: 27
		value: "message"
	}
]
```

依然像普通属性一样存储。在 [模板怎么变成真实DOM](https://djacipher.cn/2021/05/02/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/vue2/%E6%A8%A1%E6%9D%BF%E6%80%8E%E4%B9%88%E5%8F%98%E6%88%90%E7%9C%9F%E5%AE%9E%20DOM/) 中，我们讲过对字符串是按照标签来拆解解析，遇到闭合标签会调用 parseEndTag 处理，而该方法内部则会调用 closeElement 去执行结束标签解析动作，而 closeElement 又会调用 processElement 去处理还没有处理过且不存在 pre 属性的 element，处理 element 包括处理 attrs，方法是 processAttrs。

重点来看 processAttrs 方法：

```js
function processAttrs (el) {
	// 获取 attrsList
	var list = el.attrsList;
	var i, l, name, rawName, value, modifiers, syncGen, isDynamic;
	for (i = 0, l = list.length; i < l; i++) {
		// 获取到 attr 的名称和值
		name = rawName = list[i].name;
		value = list[i].value;
		// 符合 attr 名字规则继续往下执行
		if (dirRE.test(name)) {
			// mark element as dynamic
			el.hasBindings = true;
			// 这一步就是将 :parent-message.sync 进行处理，只将 sync: true 存入对象中
			modifiers = parseModifiers(name.replace(dirRE, ''));
			// support .foo shorthand syntax for the .prop modifier
			if (modifiers) {
				// 到这里 name 为 :parent-message
				name = name.replace(modifierRE, '');
			}

			// 接下来会分成三类来处理，分别是 v-bind、v-on 和 自定义 directives
			// 这里省略后两种，只保留 v-bind
			// 因为 :parent-message 是 v-bind:parent-message 的缩写形式，所以会进入下层逻辑
			if (bindRE.test(name)) { // v-bind
				// name 为 parent-message
				name = name.replace(bindRE, '');
				// value 为 message
				value = parseFilters(value);
				// 这里判断 name 是否是动态的，很显然我们这里不是动态的
				isDynamic = dynamicArgRE.test(name);
				if (isDynamic) {
					name = name.slice(1, -1);
				}
				if (
					value.trim().length === 0
				) {
					warn$2(
						("The value for a v-bind expression cannot be empty. Found in \"v-bind:" + name + "\"")
					);
				}
				if (modifiers) {
					if (modifiers.prop && !isDynamic) {
						name = camelize(name);
						if (name === 'innerHtml') { name = 'innerHTML'; }
					}
					if (modifiers.camel && !isDynamic) {
						name = camelize(name);
					}
					// 上面的直接跳过，来到这里，sync 是存在的
					if (modifiers.sync) {
						syncGen = genAssignmentCode(value, "$event");
						if (!isDynamic) {
                            // 为 update:parentMessage 添加事件
							addHandler(
								el,
								("update:" + (camelize(name))),
								syncGen,
								null,
								false,
								warn$2,
								list[i]
							);
                            // 为 update:parent-message 添加事件
							if (hyphenate(name) !== camelize(name)) {
								addHandler(
									el,
									("update:" + (hyphenate(name))),
									syncGen,
									null,
									false,
									warn$2,
									list[i]
								);
							}
						}
					}
				}
				if ((modifiers && modifiers.prop) || (
					!el.component && platformMustUseProp(el.tag, el.attrsMap.type, name)
				)) {
					addProp(el, name, value, list[i], isDynamic);
				} else {
					addAttr(el, name, value, list[i], isDynamic);
				}
			}
		}
	}
}
```

常规处理可以看代码里的注释，这里只重点说下进入 modifiers 逻辑内的两个处理：
1. syncGen = genAssignmentCode(value, "$event")
2. addHandler

先来看 genAssignmentCode

```js
function genAssignmentCode (
	value,
	assignment
) {
	var res = parseModel(value);
	if (res.key === null) {
		return (value + "=" + assignment)
	} else {
		return ("$set(" + (res.exp) + ", " + (res.key) + ", " + assignment + ")")
	}
}
```

parseModel 返回一个对象 {exp: "message", key: null}。因此这里会走第一分支返回字符串："message=$event"。

然后就是调用 addHandler 方法：addHandler(el, "update:parentMessage", "message=$event", null, false, warn, list[i])

```js
function addHandler (el, name, value, modifiers, important, warn, range, dynamic) {
	modifiers = modifiers || emptyObject;
	var events = el.events || (el.events = {});
	// 省略
	var newHandler = rangeSetItem({ value: value.trim(), dynamic: dynamic }, range);
	if (modifiers !== emptyObject) {
		newHandler.modifiers = modifiers;
	}

	var handlers = events[name];
	if (Array.isArray(handlers)) {
		important ? handlers.unshift(newHandler) : handlers.push(newHandler);
	} else if (handlers) {
		events[name] = important ? [newHandler, handlers] : [handlers, newHandler];
	} else {
		events[name] = newHandler;
	}
}
```

通过 rangeSetItem 创建一个事件处理方法，赋值给 el.events['update:parentMessage']。rangeSetItem 内部做了些什么处理呢？

```js
function rangeSetItem (item, range) {
	if (range) {
		if (range.start != null) {
			item.start = range.start;
		}
		if (range.end != null) {
			item.end = range.end;
		}
	}
	return item
}
```

传入的 item 是对象 { value: "message=$event", dynamic: null }，rangeSetItem 是给这个对象添加 start 和 end 属性，然后返回 item。

也就是说为 el 添加了一个可以监听的事件 "update:parentMessage"。 el.events['update:parentMessage'] 的值如下所示

```js
{
	dynamic: undefined
	end: 57
	start: 27
	value: "message=$event"
}
```

 processAttrs 执行完之后，依据条件，会继续执行 addAttr(el, name, value, list[i], isDynamic) 向 el.attrs push 进一个对象，即下面这个：

```js
{
	dynamic: false
	end: 57
	name: "parent-message"
	start: 27
	value: "message"
}
```

这些都处理完之后，相当于准备工作都完成了，那么按照上方的执行流程图，AST 创建完成后，会生成 render 函数，如下所示：

```js
(function anonymous(
) {
with(this){return _c('div',{attrs:{"id":"demo"}},[_c('child',{attrs:{"parent-message":message},on:{"update:parentMessage":function($event){message=$event},"update:parent-message":function($event){message=$event}}})],1)}
})
```

可以看到这一次子组件添加了 on 事件对象，那么在 createComponent 内部就会将 on 事件对象赋值给 listeners，然后存入 componentOptions，最后 componentOptions 会被传入 new Vnode() 中创建 vnode。

现在 vnode 创建成功，就可以开始进入 patch 阶段创建 Dom 了。而创建 Dom 就会会执行 child._init() 方法。_init 方法内部执行的 initInternalComponent 方法有这样一段处理：

```js
var opts = vm.$options = Object.create(vm.constructor.options);
var vnodeComponentOptions = parentVnode.componentOptions;
opts._parentListeners = vnodeComponentOptions.listeners;
```

转换过来就是这样：vm.$options._parentListeners = parentVnode.componentOptions.listeners。

在 _init 内部会继续执行 initEvents(vm)，也就是会执行 updateComponentListeners(vm, listeners)。

```js
function initEvents (vm) {
	vm._events = Object.create(null);
	vm._hasHookEvent = false;
	// init parent attached events
	var listeners = vm.$options._parentListeners;
	if (listeners) {
		updateComponentListeners(vm, listeners);
	}
}

function updateComponentListeners (vm, listeners, oldListeners) {
	target = vm;
	// add 是一个方法
	updateListeners(listeners, oldListeners || {}, add, remove, createOnceHandler, vm);
	target = undefined;
}

// 继续执行 updateListeners
function updateListeners (on, oldOn, add, remove, createOnceHandler, vm) {
	var name, def, cur, old, event;
	for (name in on) {
		def = cur = on[name];
		old = oldOn[name];
		event = normalizeEvent(name);
		if (isUndef(cur)) {
			warn(
				"Invalid handler for event \"" + (event.name) + "\": got " + String(cur),
				vm
			);
		} else if (isUndef(old)) {
			if (isUndef(cur.fns)) {
				cur = on[name] = createFnInvoker(cur, vm);
			}
			if (isTrue(event.once)) {
				cur = on[name] = createOnceHandler(event.name, cur, event.capture);
			}
			add(event.name, cur, event.capture, event.passive, event.params);
		} else if (cur !== old) {
			old.fns = cur;
			on[name] = old;
		}
	}
}
```

updateListeners 做的事情就是遍历 listeners 对象，获取到每一个事件的事件名和执行的方法，执行 add(event.name, cur, event.capture, event.passive, event.params)。

```js
function add (event, fn) {
	target.$on(event, fn);
}
```

而 add 方法就是调用 $on 去注册方法，所以我们就可以使用 $emit 去触发这个事件。

讲完了整个流程之后，需要再补充一个问题：message 改变之后触发的更新，render 函数中如何添加上 on 事件对象？

在 [模板怎么变成真实DOM](https://djacipher.cn/2021/05/02/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/vue2/%E6%A8%A1%E6%9D%BF%E6%80%8E%E4%B9%88%E5%8F%98%E6%88%90%E7%9C%9F%E5%AE%9E%20DOM/) 中，我们讲 genElement 步骤时提到过，如果是 component，会走 genComponent 生成 component render 字符串：

```js
return ("_c(" + componentName + "," + (genData(el, state)) + (children ? ("," + children) : '') + ")")
```

getData 就是获取各种属性信息包括 attrs 等。简化之后如下

```js
// attributes
if (el.attrs) {
	data += "attrs:" + (genProps(el.attrs)) + ",";
}
// event handlers
if (el.events) {
	data += (genHandlers(el.events, false)) + ",";
}
```

因为我们在前面为 el.event 添加了事件对象，因此就可以通过 genHandlers(el.events, false) 去添加 on 事件对象

```js
var prefix = isNative ? 'nativeOn:' : 'on:';
var staticHandlers = "";
var dynamicHandlers = "";
for (var name in events) {
	var handlerCode = genHandler(events[name]);
	if (events[name] && events[name].dynamic) {
		dynamicHandlers += name + "," + handlerCode + ",";
	} else {
		staticHandlers += "\"" + name + "\":" + handlerCode + ",";
	}
}
staticHandlers = "{" + (staticHandlers.slice(0, -1)) + "}";
if (dynamicHandlers) {
	return prefix + "_d(" + staticHandlers + ",[" + (dynamicHandlers.slice(0, -1)) + "])"
} else {
	return prefix + staticHandlers
}
```

prefix 是判断是否是原生事件，这里就是 on。遍历 events 对象，拿到每个事件的执行方法，因为事件名是静态的，所以会走第二分支，最后执行
> return "on:{"update:parentMessage":function($event){message=$event},"update:parent-message":function($event){message=$event}}"

这样就为子组件的 render 字符串加上了 on 属性。

## 最后

本文依据父组件传递、子组件接收、子组件渲染这样一个流程讲解了 props 渲染过程。也解析了通过特定的语法，可以在子组件内触发父组件属性的更新，进而更新 prop 的原理。
