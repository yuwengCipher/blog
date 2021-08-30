---
title: 探寻 slot 的奥秘
date: 2021-07-18
categories:
 - Vue
tags:
 - Vue
---

## 前言

slot 思想借鉴于 Web Component，使用 &lt;slot&gt; 为内容提供一个占位符，然后我们就可以往里面填充自定义内容。slot 有不同的用法形式，官方文档也有详细说明，因此本篇不是来讲解用法的，我们的任务是弄明白 slot 工作的原理。

由于 slot 的用法有很多种，每一种涉及的情形和处理也不同，但基本原理是相同的，所以本篇只会讲解具名插槽 named slots 和 作用域插槽 scoped slots。

开始之前，需提前查看 [模板怎么变成真实DOM](https://djacipher.cn/2021/05/02/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/vue2/%E6%A8%A1%E6%9D%BF%E6%80%8E%E4%B9%88%E5%8F%98%E6%88%90%E7%9C%9F%E5%AE%9E%20DOM/) 了解 Vue 大致的渲染流程，在这篇中，transform 步骤讲的是 AST 如何转变为 render 字符串，涉及到的 genELement 方法中会使用 genSlot 处理 slot。

## 具名插槽

我们将 demo 修改为如下所示：

```js
// html
<div id="demo">
	<Child>
		<template v-slot:default>{{branches[0]}}</template>
	</Child>
</div>

// js
Vue.component("Child", {
	template: '<p><slot></slot></p>'
})
new Vue({
	data: {
		branches: ['A', 'B']
	}
})
```

### genSlot

```js
function genSlot (el, state) {
	// 获取 slotName，默认为 default
	var slotName = el.slotName || '"default"';
	var children = genChildren(el, state);
	var res = "_t(" + slotName + (children ? (",function(){return " + children + "}") : '');
	// 省略
	return res + ')'
}
```

genSlot 就是将 AST 中的 slot 属性包装成 render 函数字符串，render 函数中使用 _t 方法渲染 slot 属性。此 demo 的 render 函数字符串为

> _t("default")

_t 指向 renderSlot 方法，它最终返回 vnodes。

### renderSlot

```js
function renderSlot (name, fallbackRender, props, bindObject) {
	var scopedSlotFn = this.$scopedSlots[name];
	// 省略
}
```

这里面需要用到 this.$scopedSlots，而 this.$scopedSlots 是在 Vue.prototype._render 函数内部执行 normalizeScopedSlots() 得到的

### normalizeScopedSlots

```js
var vm = this;
var ref = vm.$options;
var render = ref.render;
var _parentVnode = ref._parentVnode;

if (_parentVnode) {
	vm.$scopedSlots = normalizeScopedSlots(
		_parentVnode.data.scopedSlots,
		vm.$slots,
		vm.$scopedSlots
	);
}
```

这里面又涉及到几个属性，在说明之前，需要搞清楚子组件一个大概的处理流程：

首先在父组件页面，对 Child 组件使用 _c 方法处理，会创建它的 vnode，创建代码如下：

```js
var vnode = new VNode(
	("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
	data, undefined, undefined, undefined, context,
	{ Ctor: Ctor, propsData: propsData, listeners: listeners, tag: tag, children: children },
	asyncFactory
)
```

最后一个对象会作为 componentOptions 传入，Ctor 是构造函数 VueComponent ，children 是 branches[0]。

然后就需要渲染子组件本身了，调用的方法就是创建组件的构造函数，也就是执行 new vnode.componentOptions.Ctor(options)，然后在内部就会调用 _init(options)。入参 options 如下所示：

```js
var options = {
	_isComponent: true,
	_parentVnode: vnode,
	parent: parent
}
```

options._parentVnode 就是 Child vnode。

知道了这个处理流程后，下面就说明 normalizeScopedSlots 入参：

- _parentVnode 就是 Child vnode，那 data.scopedSlots 是被 Child 组件包裹的 slot 属性，这个后面会具体说。
- 另外两个参数 vm.$slots 和 vm.$scopedSlots 初始定义在 initRender 方法里。

### initRender

```js
// 省略
function initRender (vm) {
	var options = vm.$options;
	// parentVnode 就是子组件在父组件中的占位符，也就是 Child 组件
	var parentVnode = vm.$vnode = options._parentVnode;
	// context 就是 vue
	var renderContext = parentVnode && parentVnode.context;
	vm.$slots = resolveSlots(options._renderChildren, renderContext);
	vm.$scopedSlots = emptyObject;
}
```

vm.$scopedSlots 默认是一个空对象。vm.$slots 是执行 resolveSlots 方法得到的。这里需要弄清楚 _renderChildren 是什么

由于 Child 是 component，所以在调用 _init(options) 初始化时执行的是 initInternalComponent(vm, options)，

```js
// initInternalComponent
// 省略
var opts = vm.$options = Object.create(vm.constructor.options);
var parentVnode = options._parentVnode;
var vnodeComponentOptions = parentVnode.componentOptions;
opts._renderChildren = vnodeComponentOptions.children;
```

vnodeComponentOptions.children 指的是 Child 组件在父组件中的占位 vnode，在这里就是 branches[0] 所对应的 vnode。

那现在我们来看看 resolveSlots([vnode], vm) 的执行结果

```js
function resolveSlots (
	children,
	context
) {
	if (!children || !children.length) {
		return {}
	}
	var slots = {};
	for (var i = 0, l = children.length; i < l; i++) {
		var child = children[i];
		var data = child.data;
		if (data && data.attrs && data.attrs.slot) {
			delete data.attrs.slot;
		}
		if ((child.context === context || child.fnContext === context) &&
			data && data.slot != null
		) {
			var name = data.slot;
			var slot = (slots[name] || (slots[name] = []));
			if (child.tag === 'template') {
				slot.push.apply(slot, child.children || []);
			} else {
				slot.push(child);
			}
		} else {
			(slots.default || (slots.default = [])).push(child);
		}
	}
	return slots
}
```

先创建一个声明式对象 slots，如果 children 存在，那么就会去遍历 children
- 如果 child vnode 是作为 slot，那么就要删除 data.attrs.slot
- 如果 存在多个 named slot，只有与 Child 组件存在于同一个上下文才会被存入 slots[name] 中
- 否则就存入 slots[default] 中

由于文本 vnode 不存在 data 属性，因此这里会进入最后一个分支，为 slots 添加 default 数组并将 vnode 存入，即 slots = {default: [vnode]}。

总结来说，就是不同类型的 slot 会被存入不同的集合中。

弄明白了这些之后，我们再回过头来看看 normalizeScopedSlots。

它有三个参数，第一个 slots 是上面的 _parentVnode.data.scopedSlots，值是 undefined；第二个 normalSlots 是 resolveSlots() 的值 slots；第三个参数 vm.$scopedSlots 是一个空对象。

```js
function normalizeScopedSlots (
	slots,
	normalSlots,
	prevSlots
) {
	var res;
	var hasNormalSlots = Object.keys(normalSlots).length > 0;
	var isStable = slots ? !!slots.$stable : !hasNormalSlots;
	var key = slots && slots.$key;
	if (!slots) {
		res = {};
	} else if (slots._normalized) {
		return slots._normalized
	} else if (
		isStable &&
		prevSlots &&
		prevSlots !== emptyObject &&
		key === prevSlots.$key &&
		!hasNormalSlots &&
		!prevSlots.$hasNormal
	) {
		return prevSlots
	} else {
		res = {};
		for (var key$1 in slots) {
			if (slots[key$1] && key$1[0] !== '$') {
				res[key$1] = normalizeScopedSlot(normalSlots, key$1, slots[key$1]);
			}
		}
	}
	// expose normal slots on scopedSlots
	for (var key$2 in normalSlots) {
		if (!(key$2 in res)) {
			res[key$2] = proxyNormalSlot(normalSlots, key$2);
		}
	}
	def(res, '$stable', isStable);
	def(res, '$key', key);
	def(res, '$hasNormal', hasNormalSlots);
	return res
}
```

由于 slots 不存在，那么 res = {}，然后遍历 normalSlots，为 res 添加 slot 属性：key 是 'default'，value 方法 proxyNormalSlot。

```js
function proxyNormalSlot (slots, key) {
	return function () { return slots[key]; }
}
```

proxyNormalSlot 返回的也是一个方法，执行这个方法就会得到对应 slotName 的 vnode，在这里就是 return slots['default']。

最后是给 res 添加 $stable、$key、$hasNormal 这三个属性。

也就是说 $scopedSlots 是如下对象：

```js
$scopedSlots = {
	default: () => { return vnode},
	$stable: isStable,
	$key: key,
	$hasNormal: hasNormalSlots,
}
```

现在 $scopedSlots 对象拿到了，那么在 renderSlot 内就可以通过 this.$scopedSlots['default'] 拿到对应的 vnode，后面就是进入正常的 DOM 创建流程了。

以上，我们关注重点信息，梳理了 slot 内容是如何从父组件传递到子组件的过程，可以结合下图进行查看：

![render流程](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/render流程.png)

## 同名插槽

上面的例子中只有一个 default slot，假如存在多个同名插槽，会是一个什么样的结果呢？。

将例子稍微改动如下：有2个相同的 default 和 b 插槽。

```js
<div id="demo">
	<Child>
		<template>default: {{branches[0]}}</template><br>
		<template>default: {{branches[1]}}</template><br>
		<template v-slot:b>b: {{branches[1]}}</template><br>
		<template v-slot:b>b: {{branches[0]}}</template>
	</Child>
</div>

// js
Vue.component("Child", {
	template: '<p><slot></slot><slot name="b"></slot></p>'
})
```

经过 normalizeScopedSlots 处理之后，可以看到 $scopedSlots 是这样：

```js
$scopedSlots = {
	b: () => { return vnode},
	default: () => { return vnode},
	$stable: isStable,
	$key: key,
	$hasNormal: hasNormalSlots,
}
```

但是页面显示为:

```js
default: A
default: B

b: A
```

父组件的 AST 处理生成的 render 函数如下：

```js
(function anonymous(
) {
with(this){return _c('div',{attrs:{"id":"demo"}},[_c('child',{scopedSlots:_u([{key:"b",fn:function(){return [_v(_s(branches[0]))]},proxy:true}])},[[_v(_s(branches[0]))],_c('br'),_v(" "),[_v(_s(branches[1]))],_c('br'),_v(" "),_c('br')],2)],1)}
})
```

假如再修改成下面这样：为默认 template 添加了一个 v-slot:default

```js
<template>default: {{branches[0]}}</template><br>
<template v-slot:default>default: {{branches[1]}}</template><br>
<template v-slot:b>b: {{branches[1]}}</template><br>
<template v-slot:b>b: {{branches[0]}}</template>
```

页面显示为：

```js
default: Bb: A
```

父组件的 AST 处理生成的 render 函数如下：

```js
(function anonymous(
) {
with(this){return _c('div',{attrs:{"id":"demo"}},[_c('child',{scopedSlots:_u([{key:"default",fn:function(){return [_v("default: "+_s(branches[1]))]},proxy:true},{key:"b",fn:function(){return [_v("b: "+_s(branches[0]))]},proxy:true}])},[[_v("default: "+_s(branches[0]))],_c('br'),_v(" "),_c('br'),_v(" "),_c('br')],2)],1)}
})
```

结合以上例子的不同结果，可以得出一个结论：默认插槽多个内容会合并显示，具名插槽会以最后一个内容显示，并且如果存在具名 default，那么默认插槽的内容就会被舍弃。

那现在有两个问题，一个是同名具名插槽为什么会只保留最有一个？另一个是具名 default 为什么会导致默认 default 内容丢失？

接下来我们先看看第一个问题。

可以看到 Child 组件中具名插槽这一部分被当做 scopedSlots 存入 data 中进行处理，处理的方法是 _u，也就是 resolveScopedSlots。其他的则存入 children 中正常处理。那我们继续往前查看 AST tree。

开始之前，需要了解一点，模板字符串是以标签为单位进行分隔处理的，每一对标签处理结束时会调用 closeElement 方法处理。

步骤一：首先会执行 processElement 方法。

```js
if (!inVPre && !element.processed) {
	element = processElement(element, options);
}
```

在它的内部执行 processSlotContent(element) 处理 slot，因为我们用的新语法即使用 template 标签来包括内容，所以只看下面这部分：

```js
if (el.tag === 'template') {
	var slotBinding = getAndRemoveAttrByRegex(el, slotRE);
	if (slotBinding) {
		{
			if (el.slotTarget || el.slotScope) {
				warn$2(
					"Unexpected mixed usage of different slot syntaxes.",
					el
				);
			}
			if (el.parent && !maybeComponent(el.parent)) {
				warn$2(
					"<template v-slot> can only appear at the root level inside " +
					"the receiving component",
					el
				);
			}
		}
		var ref = getSlotName(slotBinding);
		var name = ref.name;
		var dynamic = ref.dynamic;
		el.slotTarget = name;
		el.slotTargetDynamic = dynamic;
		el.slotScope = slotBinding.value || emptySlotScopeToken;
	}
}
```

在模板中的标签被处理时，标签上的属性会被存进 AST 中的 attrList 中，那么 v-slot 属性就会被存进去。getAndRemoveAttrByRegex(el, slotRE) 就是取出 slot 属性对象 slotBinding，然后就是为自身 AST 添加 slot 相关的 slotTarget、slotScope 等属性

步骤二：为父 AST 添加 scopedSlots 属性。

processElement 处理完成后，接着会执行下面这段逻辑：

```js
if (currentParent && !element.forbidden) {
	if (element.elseif || element.else) {
		processIfConditions(element, currentParent);
	} else {
		if (element.slotScope) {
			var name = element.slotTarget || '"default"'
				; (currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element;
		}
		// 省略
	}
}
```

currentParent 是 Child，而且 template 标签中不存在 v-if 等指令，那么就会进入第二分支。

element.slotScope 和 element.slotTarget 在 processElement 处理时已经加上了。首先将 element.slotTarget 也就是 slot 的名称赋值给 name。然后就是对 currentParent.scopedSlots 的处理，如果不存在就声明为对象；如果它存在，那么就将当前 AST 作为一个属性存入，key 就是 template 的 slot 命名，比如说这种：

> currentParent.scopedSlots['b'] = bAST

概括来说，步骤一是为 AST 添加 slotTarget 和 slotScope 等属性，步骤二是将 AST 存入父组件的 scopedSlots 对象中。经过第二个步骤的处理，同名插槽会覆盖前面的内容，也就是说同名插槽只保留最后一个的内容。

现在来看第二个问题。

我们知道 render 函数中的 _u 就是 resolveScopedSlots

```js
function resolveScopedSlots (
	fns,
	res,
	hasDynamicKeys,
	contentHashKey
) {
	res = res || { $stable: !hasDynamicKeys };
	for (var i = 0; i < fns.length; i++) {
		var slot = fns[i];
		if (Array.isArray(slot)) {
			resolveScopedSlots(slot, res, hasDynamicKeys);
		} else if (slot) {
			if (slot.proxy) {
				slot.fn.proxy = true;
			}
			res[slot.key] = slot.fn;
		}
	}
	if (contentHashKey) {
		(res).$key = contentHashKey;
	}
	return res
}
```

它的第一个参数是一个数组，比如说下面这种：

```js
[{key:"default",fn:function(){return [_v("default: "+_s(branches[1]))]},proxy:true},{key:"b",fn:function(){return [_v("b: "+_s(branches[0]))]},proxy:true}]
```

在对数组的循环处理中，内部逻辑最终会进入第二个分支，执行 res[slot.key] = slot.fn，那么 res 就拥有了名称为 b 和 default 的方法。也就是说在执行 _c('child') 时， Child vnode 的 scopedSlots 是一个拥有这2个方法的对象。

我们再回到 Child 组件内部的 slot 处理方法 normalizeScopedSlots，我们看一下重点部分，简化后如下所示

```js
res = {};
for (var key$1 in slots) {
	if (slots[key$1] && key$1[0] !== '$') {
		res[key$1] = normalizeScopedSlot(normalSlots, key$1, slots[key$1]);
	}
}

for (var key$2 in normalSlots) {
	if (!(key$2 in res)) {
		res[key$2] = proxyNormalSlot(normalSlots, key$2);
	}
}
```

这里的 slots 就是 _parentVnode.data.scopedSlots，拥有 b 和 default 方法；normalSlots 是被 Child 组件包裹的 vnode。

第一个循环里，是对 normalSlots 的处理

```js
function normalizeScopedSlot (normalSlots, key, fn) {
	var normalized = function () {
		var res = arguments.length ? fn.apply(null, arguments) : fn({});
		res = res && typeof res === 'object' && !Array.isArray(res)
			? [res]
			: normalizeChildren(res);
		var vnode = res && res[0];
		return res && (
			!vnode ||
			(res.length === 1 && vnode.isComment && !isAsyncPlaceholder(vnode))
		) ? undefined
			: res
	};
	if (fn.proxy) {
		Object.defineProperty(normalSlots, key, {
			get: normalized,
			enumerable: true,
			configurable: true
		});
	}
	return normalized
}
```

normalizeScopedSlot 实际上就是返回一个方法，也就是每遍历一个 vnode，就会为 res 添加一个方法，那么如果遇到同名的，就会覆盖掉原有的。另外还重写了 normalSlots，它本来是 Child 包裹的 vnode，但是经过处理后，它也是跟 scopedSlots 类似的对象。当前的例子来说，就是将它变成拥有 b 和 default 方法的对象。也就是说默认 default 的内容被去除了。

## 作用域插槽

简单来说，作用域插槽使父组件能够获取到子组件里的属性。

我们将 demo 改成如下所示：

```js
<Child>
	<template v-slot:default="slotProps">change in parent: {{slotProps.branchesInChild[1]}}</template><br>
</Child>

Vue.component("Child", {
	data () {
		return {
			branchesInChild: ['A', 'B'],
		}
	},
	template: '<p><slot v-bind:branchesInChild="branchesInChild">render in child: {{branchesInChild[0]}}</slot></p>'
})
```

Child 组件默认应该会渲染成 render in child: A，但是我们做了修改如父组件中所示，页面会被渲染成 change in parent: B。

说明可以在父组件层面拿到子组件提供的信息去决定 Child 组件的渲染。这在日常工作中经常遇到，也非常有用，比如说需要将 table 组件中的某一个 column 渲染的信息自定义。

接下来就以此为例看看这是如何实现的吧。

首先看 processSlotContent(element) 处理 template 标签中的 slot 属性（上面有罗列代码，这里不重复了）：

它通过 getAndRemoveAttrByRegex 拿到 slotBinding，其中就包括我们绑定的 default=slotProps，name 是 'default'，value 是 'slotProps'。将 'default' 赋值给 element.slotTarget、 'slotProps' 赋值给 element.slotScope。

我们来看看 parnet 组件的 render 函数

```js
(function anonymous(
) {
with(this){return _c('div',{attrs:{"id":"demo"}},[_c('child',{scopedSlots:_u([{key:"default",fn:function(slotProps){return [_v("change in parent: "+_s(slotProps.branchesInChild[1]))]}}])},[_c('br')])],1)}
})
```

可以看到 scopedSlots 中的处置方法现在需要接收 slotProps 对象用作渲染指定属性，而且渲染的就是我们在父组件中设置的值 change in parent: {{slotProps.branchesInChild[1]}}。

依据上面 normalizeScopedSlots 方法的解析我们知道，vm.$scopedSlots.default 就会是封装这个 fn 的 normalized 方法。它在内部会接收参数，作为入参调用 fn。

```js
var res = arguments.length ? fn.apply(null, arguments) : fn({});
```

那现在就需要看看这个 slotProps 对象是如何传进去的。

先看看 Child 组件内部的 render 函数

```js
(function anonymous(
) {
with(this){return _c('p',[_t("default",function(){return [_v("render in child: "+_s(branchesInChild[0]))]},{"branchesInChild":branchesInChild})],2)}
})
```

_t 是 renderSlot，三个参数分别是 fnKey、fnVal 和 {"branchesInChild":branchesInChild}。这里的重点就是多了一个 {"branchesInChild":branchesInChild} 这个对象，它会作为 props 传入 renderSlot 中，用于通过 this.$scopedSlots 中的方法获取 vnodes。

```js
function renderSlot (
	name,
	fallbackRender,
	props,
	bindObject
) {
	var scopedSlotFn = this.$scopedSlots[name];
	var nodes;
	if (scopedSlotFn) {
		props = props || {};
		if (bindObject) {
			if (!isObject(bindObject)) {
				warn('slot v-bind without argument expects an Object', this);
			}
			props = extend(extend({}, bindObject), props);
		}
		nodes =
			scopedSlotFn(props) ||
			(typeof fallbackRender === 'function' ? fallbackRender() : fallbackRender);
	} else {
		nodes =
			this.$slots[name] ||
			(typeof fallbackRender === 'function' ? fallbackRender() : fallbackRender);
	}

	var target = props && props.slot;
	if (target) {
		return this.$createElement('template', { slot: target }, nodes)
	} else {
		return nodes
	}
}
```

简单梳理一下处理逻辑：

- 首先获取到 vm.$scopedSlots['default'] 方法。这个是存在的，进入第一个分支。
  - props 就是第三个对象参数，bindObject 不存在，那么就直接执行 scopedSlotFn(props)，将 branchesInChild 传入了 default 方法中去获取 vnodes，赋值给 nodes
- props.slot 不存在，直接返回 nodes。

就这样在 renderSlot 方法内部将 branchesInChild 信息作为入参，使用 vm.$scopedSlots['default'] 方法获取到了 Child 组件包裹的 vnodes，然后将 vnodes 创建为 DOM 渲染到了页面上，实现了在父组件拿到子组件指定的信息，然后渲染自定义信息的功能。

## 最后

本篇主要了解了 slot 渲染的原理以及 scoped slot 实现的原理，总结如下：

1. 默认插槽其实跟一般的具名插槽一样，因为它会被默认为 default 插槽
2. 父组件针对 template 上的插槽名称会创建对应方法存于 scopedSlots，执行这个方法就会返回被 template 包裹的内容 vnodes。如果存在同名的，只会保留后一个的内容。然后子组件内部渲染时会去找插槽名对应的方法去渲染。
3. 父组件中 template 的 v-slot:default="slotProps" 在被转成 render 字符串时会给方法添加一个可以接收的参数 slotProps;子组件中 slot 的 v-bind:branchesInChild="branchesInChild" 在被转成 render 字符串时会给 _t 方法添加一个 props 参数 {"branchesInChild":branchesInChild}。这样就可以将 props 传入方法中去获取 vnodes。
