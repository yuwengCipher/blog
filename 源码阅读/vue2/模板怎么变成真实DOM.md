---
title: 模板怎么变成真实DOM
date: 2021-04-19
categories:
 - Vue
tags:
 - Vue
---

## 前言

官方文档有过这样一段描述：
> Vue.js 的核心是一个允许采用简洁的模板语法来声明式地将数据渲染进 DOM 的系统

其实 Vue 模板语法与 ejs 等模板语言类似，基本思想就是： template + data = dom。那 Vue 是如何来将模板语法结合数据转译成真实 Dom 呢？本章将会来探究这个秘密。

了解过编译和转译的应该知道，它们大致都会经历 parse、transform、generate 三个步骤。不同的是，编译通常来说是高级语言向低级语言的变换，而转译则只是语法层面的转换，Vue 的模板向 DOM 的变化就是转译。除了上述三个步骤之外，最后 DOM 的生成还需要 patch 操作。

开始之前，有必要梳理一下转译过程涉及到的一些重要方法（如下所示）调用及它们之间的关系，做到心中有数，将关注点放在转译步骤上。

```js
Vue.prototype.$mount = function (){}
function parse(template, options) {};
function generate (ast,options) {}

function createCompilerCreator(baseCompile) {
	return function createCompiler (baseOptions) {
		function compile(template, options) {
			var compiled = baseCompile(template.trim(), finalOptions);
			return compiled
		};
		return {
			compile: compile,
			compileToFunctions: createCompileToFunctionFn(compile)
		}
	}
};

function createCompileToFunctionFn (compile) {
	return function compileToFunctions (template, options, vm) {
		var compiled = compile(template, options);
		res.render = createFunction(compiled.render, fnGenErrors);
		return res;
	}
	
}

function baseCompile(template, options) {
	var ast = parse(template.trim(), options);
	return {
		ast: ast,
		render: code.render,
		staticRenderFns: code.staticRenderFns
	}
};

```

大致的执行顺序及关系是：

1. var createCompiler = createCompilerCreator(baseCompile)
2. var ref$1 = createCompiler(baseOptions);
3. var compileToFunctions = ref$1.compileToFunctions
4. $mount();
5. var ref = compileToFunctions(template, options)
6. var compiled = compile(template, options)
7. compiled = baseCompile(template.trim(), finalOptions)
8. compiled.ast = parse(template.trim(), options)
9. compiled.render = generate(ast, options).render
10. ref.render = createFunction(compiled.render, fnGenErrors)
11. Vue.$options.render = ref.render
12. mount.call(this, el, hydrating)

## 转译步骤

### parse

这一步将 template 作为字符串进行特定规则分析转换成 AST。

从上面的梳理结果可以看到，真正的 parse 动作从 parse(template.trim(), options) 开始，所以我们从这里开始。

```js
function parse(template, options) {
	// 省略代码...
	var stack = [];
	var preserveWhitespace = options.preserveWhitespace !== false;
	var whitespaceOption = options.whitespace;
	// root 就是最后的 AST 结果
	var root;
	var currentParent;
	var inVPre = false;
	var inPre = false;
	var warned = false;
	// 省略代码...
	parseHTML(template, {})
};
```

核心是 parseHTML 方法，简单来说就是通过 while 循环，依据标签作为分隔标识将 template 分隔成多个子字符串进行处理。

因为 template 也是标签，所以选择标签为单位进行对应的 parse 操作；而分隔之后的字符串结构如下图所示，大家一定觉得很眼熟，对，就是多叉树，这个循环过程实际上就是按深度优先原则对多叉树的遍历。

![templateTree](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/templateTree.png)

每一个节点就是一个标签、文本字符、空格或者是注释，parseHTML 接受两个参数：template、options，其中 options 拥有 start、end、chars、comment 方法，分别用来处理开始标签、结束标签、文本字符、注释。接下来大致说下整个流程

首先在循环体内部有两大分支，第一大分支处理正常情况（除了 script、style、textarea 标签），第二个分支就是处理特殊情况，我们先看正常情况的处理操作。

- 在当前给定的字符串中查找第一个 '<' 的下标
  - 如果下标是 0，处理方式如下：
    - 普通注释, '-->' 结尾，如果 options.shouldKeepComment 为 true，则需要保留注释。调用 advance(length)
    - 特殊注释，']>' 结尾，调用 advance(length)
    - DOCTYPE 标识，调用 advance(length)
    - 结束标签，调用 advance(length)，parseEndTag 处理结束标签
    - 开始标签，handleStartTag 处理开始标签

> advance(length) 的作用就是以 length 为开始点截取 html 剩余部分。

重点看看 handleStartTag 和 parseEndTag。

```js
var startTagMatch = parseStartTag();
if (startTagMatch) {
	handleStartTag(startTagMatch);
}
```

parseStartTag() 方法返回一个对象：

```js
let match = {
	// 一个二维数组，内部每一个数组保存开始标签上的一个属性
	// 内数组形如 [" id=\"demo\"", "id", "=", "demo", start: 4, end: 14]
	// start 为属性开始下标，end 为属性结束下标
	attrs: Array,
	// 标签结束下标
	end: Number,
	// 标签开始下标
	start: Number,
	// 标签名称
	tagName: String,
	// 单标签斜线
	unarySlash: String
}
```

弄清楚了 match 的内容，就可以继续去看看 handleStartTag 方法了。我们主要看后半部分的逻辑。

```js
for (var i = 0; i < l; i++) {
	var args = match.attrs[i];
	var value = args[3] || args[4] || args[5] || '';
	var shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
		? options.shouldDecodeNewlinesForHref
		: options.shouldDecodeNewlines;
	attrs[i] = {
		name: args[1],
		value: decodeAttr(value, shouldDecodeNewlines)
	};
	if (options.outputSourceRange) {
		attrs[i].start = args.start + args[0].match(/^\s*/).length;
		attrs[i].end = args.end;
	}
}

if (!unary) {
	stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs, start: match.start, end: match.end });
	lastTag = tagName;
}

if (options.start) {
	options.start(tagName, attrs, unary, match.start, match.end);
}
```

声明一个与 match.attrs 长度相同的数组 attrs，通过遍历 attrs，创建一个新对象，对每一项做如下处理：
- 将属性名赋值给 name 属性，属性值赋值给 value
- match.attrs.start 去除空格后赋值个 start，end 赋值给 end
最后的 attrs 表现如下：

```js
[
	{
		name: "id",
		value: "demo",
		start: 5,
		end: 14,
	}
]
```

上面有声明过 unary，它的作用就是判断当前标签是否是单标签，如果不是单标签，那么就将该标签相关信息用对象包裹，存入 stack 中。

接着，会调用 options.start(tagName, attrs, unary, match.start, match.end), 我们找到 start 方法：

```js
start: function start (tag, attrs, unary, start$1, end) {}
```

在源码中可以看到，最开始调用 createASTElement(tag, attrs, currentParent) 创建 AST 对象，后面就是对 element 的一些处理:
1. 添加相关属性、
2. 处理 v-for, v-if, v-once 指令、
3. 是否需要将 element 赋值给 root、
4. 如果不是单标签，将 element 赋值给 currentParent，并且将 element 存入 stack(parse 方法下的 stack)，反之则调用 closeElement 方法结束标签处理操作。

后面的一些处理细节暂不做探究，主要看看 AST 的数据结构。

```js
function createASTElement (tag, attrs, parent) {
	return {
		type: 1,
		tag: tag,
		attrsList: attrs,
		attrsMap: makeAttrsMap(attrs),
		rawAttrsMap: {},
		parent: parent,
		children: []
	}
}
```

parent 与 父标签建立联系，children 可以用来存储多个子标签或子元素。其中 type 的值是1，也就是说标签的类型都是1，那文本或者其他类型呢？我们在源码中找到 options.chars 方法。

```js
if (!inVPre && text !== ' ' && (res = parseText(text, delimiters))) {
	child = {
		type: 2,
		expression: res.expression,
		tokens: res.tokens,
		text: text
	};
} else if (text !== ' ' || !children.length || children[children.length - 1].text !== ' ') {
	child = {
		type: 3,
		text: text
	};
}
```

上面代码表示出了 text 字符串最终有可能转换成 type 为2和3的 AST：
type 为2的情况（同时满足）：
- 不处于 v-pre 指令标签下
- text !== ' '
- parseText 是判断 text 是否是插值语法

type 为3的情况（满足其一即可）：
- text !== ' '
- 父标签没有子元素
- 父标签最后一个子元素的值 !== ' '

这里需要说的一点是，type 为3的第三个条件，因为一般换行符都会转换成空格符 ' ', 为了保证只存在一个换行符，才需要在已经存在子元素的情况下检验最后一个子元素是否是换行符。

通过上面我们知道了 node 元素类型的处理，现在说说模板指令。Vue 模板涉及到各种指令，增加了非常好用的功能，那这里就简单了解一下 v-for, v-if, v-once。这三个指令的处置在 options.start 方法内。
开始之前先在 demo 基础上加上一段代码以便调试。

```js
<div id="demo">
	<p v-once>测试 v-once</p>
	<div v-for="record in commits" v-if="commits.length > 0">
		<span>{{ record.commit.message }}</span>
	</div>
	...
</div>
```

先来说说对 v-for 的处理：
- var exp = getAndRemoveAttr(el, 'v-for')。从 attrsList 中获取到 v-for 对应的值 'record in commits'
- var res = parseFor(exp)。将 record 和 commits 分别存入 alias 和 for， 即{alias: record, for: commits}
- extend(el, res)。将 alias 和 for 属性存入 element 中

v-if：
- var exp = getAndRemoveAttr(el, 'v-if')。从 attrsList 中获取到 v-if 对应的值 'commits.length > 0'
- 如果 exp 存在
  - 为 el 添加 if 属性，值为 exp
  - addIfCondition(el, {exp: exp, block: el})。
- 如果 exp 不存在
  - 如果 v-else 存在，为 el 添加 else 属性，值为 true
  - var elseif = getAndRemoveAttr(el, 'v-else-if')。 如果 elseif 存在，为 el 添加 elseif 属性，值为 elseif

v-once: var once = getAndRemoveAttr(el, 'v-once')。 如果 once 存在，为 el 添加 once 属性，值为 true

也就是说，指令的处理其实是将指令对应的属性添加到 AST 对象中。到这里，parse 转换生成的 AST 雏形已经出来了，root 对象一旦确定，后面的子标签对象都会存入 children，而子标签也会存在 children，因此 AST 就是这样一棵树。

![AST](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/AST.png)

### transform

在明白 template 到 AST 的变化步骤之后，就可以继续探索 AST 是如何生成 render 函数方法。
其实这一步主要是将 AST 转换成 render str 函数字符串，然后再通过 new Function(render str) 生成函数方法。由于函数字符串的转换涉及到的情况很多，过程非常复杂，因此这里只是解析 render 字符串形成的原理。

> compiled.render = generate(ast, options).render

generate 方法如下：

```js	
var state = new CodegenState(options);
var code = ast ? (ast.tag === 'script' ? 'null' : genElement(ast, state)) : '_c("div")';
return {
	render: ("with(this){return " + code + "}"),
	staticRenderFns: state.staticRenderFns
}
```

如果不存在 ast， 那么code 就是 '_c("div")'，如果存在，则需要判断是否标签名是否是 script，如果不是就走 genElement(options, state) 生成，反之，code 就是 'null'。

```js
function genElement (el, state) {
	if (el.parent) {
		el.pre = el.pre || el.parent.pre;
	}

	if (el.staticRoot && !el.staticProcessed) {
		return genStatic(el, state)
	} else if (el.once && !el.onceProcessed) {
		return genOnce(el, state)
	} else if (el.for && !el.forProcessed) {
		return genFor(el, state)
	} else if (el.if && !el.ifProcessed) {
		return genIf(el, state)
	} else if (el.tag === 'template' && !el.slotTarget && !state.pre) {
		return genChildren(el, state) || 'void 0'
	} else if (el.tag === 'slot') {
		return genSlot(el, state)
	} else {
		// component or element
		var code;
		if (el.component) {
			code = genComponent(el.component, el, state);
		} else {
			var data;
			if (!el.plain || (el.pre && state.maybeComponent(el))) {
				data = genData(el, state);
			}

			var children = el.inlineTemplate ? null : genChildren(el, state, true);
			code = "_c('" + (el.tag) + "'" + (data ? ("," + data) : '') + (children ? ("," + children) : '') + ")";
		}
		// module transforms
		for (var i = 0; i < state.transforms.length; i++) {
			code = state.transforms[i](el, code);
		}
		return code
	}
}
```

genElement 执行的步骤如下：
- 遇到 staticRoot、v-once、v-for、v-if、template 标签、slot 标签，先走对应逻辑
- 如果是 component，则生成 component 的 render 字符串
- 否则，分两步生成最终的 render 字符串
  - 调用 genData，处理 element 的各种属性：key、ref、pre、component、attrs、props、events、nativeEvents、slots、model，staticClass、staticStyle，返回属性字符串，形如："{attrs:{"id":"demo"}}"
  - 调用 genChildren 生成 children 的 render 字符串
  - code = "_c('" + (el.tag) + "'" + (data ? ("," + data) : '') + (children ? ("," + children) : '') + ")"; 将 data 与 children 进行拼装
- module transform

涉及到的主要方法如下：

```js
function genNode (node, state) {
	if (node.type === 1) {
		return genElement(node, state)
	} else if (node.type === 3 && node.isComment) {
		return genComment(node)
	} else {
		return genText(node)
	}
}

function genChildren (){
	// 省略
	var gen = altGenNode || genNode;
	return ("[" + (children.map(function (c) { return gen(c, state); }).join(',')) + "]" + (normalizationType$1 ? ("," + normalizationType$1) : ''))
}
```

也就是说，在生成 render 字符串时，children 需要用中括号包裹起来，在内部进行遍历，使用 genNode 处理每一项，也就是对不同类型采用不同的处理方法，其中文本和注释的处理很简单，而标签类型依然会重新走 genElement 方法。通过这种递归方式，从上到下调用，从下至上将所有的 AST 对象采取不同的处理，最终会生成如下 render 字符串：

```js
"with(this){return _c('div',{attrs:{\"id\":\"demo\"}},[_c('p',[_v(_s(currentBranch))]),_v(\" \"),_m(0),_v(\" \"),_l((commits),function(record){return (commits.length > 0)?_c('div',[_c('span',[_v(_s(record.commit.message))])]):_e()}),_v(\" \"),_c('h1',[_v(\"Latest Vue.js Commits\")]),_v(\" \"),_l((branches),function(branch){return [_c('input',{directives:[{name:\"model\",rawName:\"v-model\",value:(currentBranch),expression:\"currentBranch\"}],attrs:{\"type\":\"radio\",\"id\":branch,\"name\":\"branch\"},domProps:{\"value\":branch,\"checked\":_q(currentBranch,branch)},on:{\"change\":function($event){currentBranch=branch}}}),_v(\" \"),_c('label',{attrs:{\"for\":branch}},[_v(_s(branch))])]}),_v(\" \"),_c('p',[_v(\"vuejs/vue@\"+_s(currentBranch))]),_v(\" \"),_c('ul',_l((commits),function(record){return _c('li',[_c('a',{staticClass:\"commit\",attrs:{\"href\":record.html_url,\"target\":\"_blank\"}},[_v(_s(record.sha.slice(0, 7)))]),_v(\"\\n        - \"),_c('span',{staticClass:\"message\"},[_v(_s(_f(\"truncate\")(record.commit.message)))]),_c('br'),_v(\"\\n        by \"),_c('span',{staticClass:\"author\"},[_c('a',{attrs:{\"href\":record.author.html_url,\"target\":\"_blank\"}},[_v(_s(record.commit.author.name))])]),_v(\"\\n        at \"),_c('span',{staticClass:\"date\"},[_v(_s(_f(\"formatDate\")(record.commit.author.date)))])])}),0)],2)}"
```

字符串中存在的 _c、_v、_s、_f，是创建不同 vnode 的方法，可以在源码中找到，在 renderMixin 中调用了 installRenderHelpers(Vue.prototype)，所以这些方法挂载在了 Vue 上。

```js
vm._c = function (a, b, c, d) { return createElement(vm, a, b, c, d, false); };
function installRenderHelpers (target) {
	target._o = markOnce;
	target._n = toNumber;
	target._s = toString;
	target._l = renderList;
	target._t = renderSlot;
	target._q = looseEqual;
	target._i = looseIndexOf;
	target._m = renderStatic;
	target._f = resolveFilter;
	target._k = checkKeyCodes;
	target._b = bindObjectProps;
	target._v = createTextVNode;
	target._e = createEmptyVNode;
	target._u = resolveScopedSlots;
	target._g = bindObjectListeners;
	target._d = bindDynamicKeys;
	target._p = prependModifier;
}
```

待所有 AST 处理完成后，render 字符串就生成了, 使用 createFunction(compiled.render, fnGenErrors)，也就是执行 new Function(compiled.render) 将字符串转换成匿名函数，然后将 render 函数赋值给 Vue.$options.render, 这样就完成了 render 函数的创建及挂载工作。

### generate

上一步我们拿到了 render 函数，那这一步就是了解如何使用这个函数。前言罗列的最后一步是调用 mount 方法，最终指向 mountComponent 方法。

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

将 updateComponent 设定为订阅者的 getter，在初始时就会调用 updateCompoennt，触发页面第一次渲染更新。这里需要注意的是，订阅者的 cb 是 noop，因为只执行 updateComponent 就能实现页面的更新（后面会讲到），因此不需要额外的回调函数。

vm._update 方法接受的第一个参是 vnode，也就是说 vm._render() 返回的是一个 vnode，找到 _render 方法，关键的就是三行代码：

```js
var ref = vm.$options;
var render = ref.render;
vnode = render.call(vm._renderProxy, vm.$createElement);
```

这里的 render 方法就是上面说到的匿名函数，美化后如下所示：

```js
(function anonymous() {
	with(this) {
		return _c('div', {
				attrs: {
						"id": "demo"
				}
		}, [_c('p', [_v(_s(currentBranch))]), _v(" "), _m(0), _v(" "), _l((commits), function (record) {
				return (commits.length > 0) ? _c('div', [_c('span', [_v(_s(record.commit.message))])]) : _e()
		}), _v(" "), _c('h1', [_v("Latest Vue.js Commits")]), _v(" "), _l((branches), function (branch) {
				return [_c('input', {
						directives: [{
								name: "model",
								rawName: "v-model",
								value: (currentBranch),
								expression: "currentBranch"
						}],
						attrs: {
								"type": "radio",
								"id": branch,
								"name": "branch"
						},
						domProps: {
								"value": branch,
								"checked": _q(currentBranch, branch)
						},
						on: {
								"change": function ($event) {
										currentBranch = branch
								}
						}
				}), _v(" "), _c('label', {
						attrs: {
								"for": branch
						}
				}, [_v(_s(branch))])]
		}), _v(" "), _c('p', [_v("vuejs/vue@" + _s(currentBranch))]), _v(" "), _c('ul', _l((commits), function (record) {
				return _c('li', [_c('a', {
						staticClass: "commit",
						attrs: {
								"href": record.html_url,
								"target": "_blank"
						}
				}, [_v(_s(record.sha.slice(0, 7)))]), _v("\n        - "), _c('span', {
						staticClass: "message"
				}, [_v(_s(_f("truncate")(record.commit.message)))]), _c('br'), _v("\n        by "), _c('span', {
						staticClass: "author"
				}, [_c('a', {
						attrs: {
								"href": record.author.html_url,
								"target": "_blank"
						}
				}, [_v(_s(record.commit.author.name))])]), _v("\n        at "), _c('span', {
						staticClass: "date"
				}, [_v(_s(_f("formatDate")(record.commit.author.date)))])])
		}), 0)], 2)
	}
})
```

render 方法执行的方式是从下至上，下层执行的结果就是上一层的 children

目前涉及到的几种处置场景如下：
- 标签，调用 _c，传入标签名、属性、子元素
- 文本，调用 _v，对涉及到使用变量的部分则需要在内部额外调用 _s 进行处理
- v-for，调用 _l，传入需遍历的数组、遍历时执行的方法
- v-once，调用 _m
- v-if 相反情况，调用 _e
- 过滤标识 |，调用 _f

执行完毕最终会生成这样一个嵌套的对象：

![vnode](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/vnode.png)

但是有一点需要注意的是，初始时如果不满足条件的不会创建成 vnode，比如，v-if="commits.length > 0" 才会显示的 div，则没有生成对应的 vnode 存入 父元素的 children 中。

### patch

vnode 创建完成后，就可以执行 _update 方法了，实际上就是执行 patch 操作，只是初始渲染时不需比较操作，因此初始渲染和后续更新传入的参不一样。在浏览器端就是调用 createPatchFunction({ nodeOps: nodeOps, modules: modules }) 创建的，接下来看看 patch 的魔力吧。注意一点，我们这里只是探究 vnode 是如何变成 dom，不是探究 patch 过程，这两个方向不一样 :)。

```js
Vue.prototype._update = function (vnode, hydrating) {
	// initial render
	vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
}
// patch
return function patch (oldVnode, vnode, hydrating, removeOnly) {
	// 省略
	var isRealElement = isDef(oldVnode.nodeType);
	if (!isRealElement && sameVnode(oldVnode, vnode)) {
		// patch existing root node
		patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly);
	} else {
		// initrender
		if (isRealElement) {
			// 省略
			oldVnode = emptyNodeAt(oldVnode);
		}

		// replacing existing element
		var oldElm = oldVnode.elm;
		var parentElm = nodeOps.parentNode(oldElm);

		// create new node
		createElm(
			vnode,
			insertedVnodeQueue,
			// extremely rare edge case: do not insert if old element is in a
			// leaving transition. Only happens when combining transition +
			// keep-alive + HOCs. (#4590)
			oldElm._leaveCb ? null : parentElm,
			nodeOps.nextSibling(oldElm)
		);
	}
}
```

逻辑很清晰，我们传入的是 vm.$el，所以走第二个分支，依据 $el 创建一个空 vnode。紧接着是声明 oldElm 和 parentElm，分别是 #demo 和 body，用作后面方法的入参部分。最后就开始执行创建 elment 步骤了。

createElm 方法里主要是分成三类进行创建，对于标签，使用 createElement，另外还会使用 createChildren 创建子元素对应的 dom，注释使用 createComment，其他的使用 createTextNode。创建结束后执行 insert(parentElm, vnode.elm, refElm), 将创建的 dom 插入到指定的位置。

createElement 其实就是调用 document.createElement(tagName)，createComment 是 document.createComment(text)，同样的，createTextNode 是调用 document.createTextNode(text)。那子元素是怎么创建的呢？我们主要来看看 createChildren 方法。

```js
function createChildren (vnode, children, insertedVnodeQueue) {
	if (Array.isArray(children)) {
		{
			checkDuplicateKeys(children);
		}
		for (var i = 0; i < children.length; ++i) {
			createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i);
		}
	} else if (isPrimitive(vnode.text)) {
		nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)));
	}
}
```

如果包含多个 child，在检查 children 是否存在重复 key 之后，就会遍历 children，调用 createElm 处理每一个子元素；如果 text 是原生属性，则创建完 textNode 之后，直接 append 到当前位置。

有没有发现，上面说到的只是调用原生创建 dom 的方法，而通过标签生成的 dom 没有我们所赋予的各种属性、指令信息，这样的 dom 是没有灵魂的。所以必须还有一个添加灵魂的步骤。在插入之前有这样的处理：

```js
if (isDef(data)) {
	invokeCreateHooks(vnode, insertedVnodeQueue);
}
```

vnode 的所有 props、directives、events 等都存储在 data 中，那么这个 invokeCreateHooks 就是给我们的 dom 润色。

```js
function invokeCreateHooks (vnode, insertedVnodeQueue) {
	for (var i = 0; i < cbs.create.length; ++i) {
		cbs.create[i](emptyNode, vnode);
	}
	i = vnode.data.hook; // Reuse variable
	if (isDef(i)) {
		if (isDef(i.create)) { i.create(emptyNode, vnode); }
		if (isDef(i.insert)) { insertedVnodeQueue.push(vnode); }
	}
}
```

咋一看是不是觉得摸不着头脑，完全不知道在干什么。关键点是需要弄清楚 cbs.create 是个什么东西。上面说过，我们调用的 patch 方法是通过 createPatchFunction(backend) 方法创建的，这个方法除了返回 patch 和 其他一些方法外，还有对 cbs 做了处理，我将 backend.modules 相关属性整理如下，方便查看：

```js
// 这里只是注明每一个属性它所拥有的方法，方便下面 modules 数组查看
var modules = {
	attrs: {
		create: updateAttrs,
		update: updateAttrs
	},
	klass: {
		create: updateClass,
		update: updateClass
	},
	events: {
		create: updateDOMListeners,
		update: updateDOMListeners
	},
	domProps: {
		create: updateDOMProps,
		update: updateDOMProps
	},
	style: {
		create: updateStyle,
		update: updateStyle
	},
	transition: inBrowser ? {
		create: _enter,
		activate: _enter,
		remove: function remove$$1 (vnode, rm) {
			/* istanbul ignore else */
			if (vnode.data.show !== true) {
				leave(vnode, rm);
			} else {
				rm();
			}
		}
	} : {},

	ref: {
		create: function create (_, vnode) {
			registerRef(vnode);
		},
		update: function update (oldVnode, vnode) {
			if (oldVnode.data.ref !== vnode.data.ref) {
				registerRef(oldVnode, true);
				registerRef(vnode);
			}
		},
		destroy: function destroy (vnode) {
			registerRef(vnode, true);
		}
	},
	directives: {
		create: updateDirectives,
		update: updateDirectives,
		destroy: function unbindDirectives (vnode) {
			updateDirectives(vnode, emptyNode);
		}
	}
}

var backend = 
	modules: [
		attrs,
		klass,
		events,
		domProps,
		style,
		transition,
		ref,
		directives
	]
}

var hooks = ['create', 'activate', 'update', 'remove', 'destroy'];

let i, j
const cbs = {}

const { modules, nodeOps } = backend

for (i = 0; i < hooks.length; ++i) {
	cbs[hooks[i]] = []
	for (j = 0; j < modules.length; ++j) {
		if (isDef(modules[j][hooks[i]])) {
			cbs[hooks[i]].push(modules[j][hooks[i]])
		}
	}
}
```

依据上面的执行逻辑，cbs 就变成下面这样：

```js
cbs = {
	create: [updateAttrs, updateClass, updateDOMListeners, updateDOMProps, updateStyle, _enter, ref.create, updateDirectives],
	activate: [_enter],
	update: [updateAttrs, updateClass, updateDOMListeners, updateDOMProps, updateStyle, ref.update, updateDirectives],
	remove: [transition.remove],
	destroy: [ref.destroy, directives.destroy],
}
```

综合以上整理的情况我可以总结如下：
1. 不同的属性拥有针对不同情况的处理方法
2. attrs、class、domListeners、domProps、style、directives 等属性创建和更新采用的是同一个方法
3. transition 没有更新方法，但是有进入和销毁方法
4. 指令也有销毁方法

回到上面的 invokeCreateHooks，因为我们这一步是创建初始 dom，所以只需执行 cbs.create 里的方法。那么接下来，我们来看看 cbs.create 里的方法，这里只挑选几个我觉得有意思的做解析（省略不重要代码）。原来的 demo 会做一下改变，以便适应不同的调试场景

#### updateAttrs

```js
var key, cur, old;
var elm = vnode.elm;
var oldAttrs = oldVnode.data.attrs || {};
var attrs = vnode.data.attrs || {};

for (key in attrs) {
	cur = attrs[key];
	old = oldAttrs[key];
	if (old !== cur) {
		setAttr(elm, key, cur, vnode.data.pre);
	}
}

for (key in oldAttrs) {
	if (isUndef(attrs[key])) {
		if (isXlink(key)) {
			elm.removeAttributeNS(xlinkNS, getXlinkProp(key));
		} else if (!isEnumeratedAttr(key)) {
			elm.removeAttribute(key);
		}
	}
}
```

第一步，attr 的旧值与新值如果不同，则将值设置成最新的；
第二步，一个是 xlink 属性，另一个是无法枚举属性，也就是除了 contenteditable, draggable, spellcheck 之外的属性，这些属性需要被移除

#### updateClass

```js
<div id="demo">
	<p class="currentBranchClass" :class="currentBranch">{{ currentBranch }}</p>
</div>
```

```js
var data = vnode.data
var cls = genClassForVnode(vnode);

var transitionClass = el._transitionClasses;
if (isDef(transitionClass)) {
	cls = concat(cls, stringifyClass(transitionClass));
}

if (cls !== el._prevClass) {
	el.setAttribute('class', cls);
	el._prevClass = cls;
}
```

依据 demo，currentBranch 当前的值是 master，因此 vnode.data = {class: 'master', staticClass: "currentBranchClass"}, 最后生成的 cls = 'currentBranchClass master'。如果存在 transitionClass，则将它合并到 cls 中。最后就是调用原生方法添加 class 属性，并且把 cls 赋值给元素的 _preClass，相当于旧 class，用作下次判断如果新旧 class 相等，则不需要执行添加 class 操作。

#### updateDOMListeners

```js
<div id="demo">
	<input type="text" @click="testClick" v-model="currentBranch">
</div>

methods: {
	testClick() {
		console.log('yeah, you click it!')
	}
}
```

```js
var on = vnode.data.on || {};
var oldOn = oldVnode.data.on || {};
target = vnode.elm;
normalizeEvents(on);
updateListeners(on, oldOn, add, remove, createOnceHandler, vnode.context);
target = undefined;
```

因为 v-model 包含有 input 事件，所以这里的 on = {click: testClick, input: f}, 然后是使用 normalizeEvents 处理 type="range" 这种情况，最后就是更新事件到 dom 上。先来看下这个特殊情况的处理，假如我们把 input 的 type 设置成 range，那么 on = {click: testClick, _r: f}，可以看到 v-model 中的 input 事件被 range 事件覆盖了。所以 normalizeEvents 就是来解决这个问题。

```js
if (isDef(on[RANGE_TOKEN])) {
	var event = isIE ? 'change' : 'input';
	on[event] = [].concat(on[RANGE_TOKEN], on[event] || []);
	delete on[RANGE_TOKEN];
}
```

RANGE_TOKEN 即 _r，意思就是，如果是 IE，就使用 range 的 change 事件，否则使用 v-model 的 input 事件。

updateListeners 的入参分别是 当前事件集合、旧事件集合、添加事件方法、移除事件方法、创建一次事件方法、上下文对象。知道了这些，我们继续进入内部探寻，大体上就是分成两个步骤，分别是循环处理新事件和处理旧事件。处理新事件分成三个分支，第一个是处理边界，第二个是不存在旧事件就添加新事件，第三分支是替换新事件。

```js
for (name in on) {
	// 省略
	event = normalizeEvent(name);
	if (isUndef(cur.fns)) {
		cur = on[name] = createFnInvoker(cur, vm);
	}
	add(event.name, cur, event.capture, event.passive, event.params);
	// 省略
}
```

normalizeEvent(name) 处理事件修饰符，包括 capture、passive 等。接着来看 createFnInvoker(cur, vm)

```js
function invoker () {
	var arguments = arguments;

	var fns = invoker.fns;
	if (Array.isArray(fns)) {
		var cloned = fns.slice();
		for (var i = 0; i < cloned.length; i++) {
			invokeWithErrorHandling(cloned[i], null, arguments, vm, "v-on handler");
		}
	} else {
		// return handler return value for single handlers
		return invokeWithErrorHandling(fns, null, arguments, vm, "v-on handler")
	}
}
invoker.fns = fns;
return invoker
```

将 invoker 方法返回，替换了原来绑定的事件方法。fns 是原来的事件方法，它有可能是数组，也可能是单个方法，这决定了走不同的处理分支。但最终都会是执行 invokeWithErrorHandling 方法，传入原事件方法及当前接受到的参数。

```js
function invokeWithErrorHandling (handler, context, args, vm, info) {
	var res;
	try {
		res = args ? handler.apply(context, args) : handler.call(context);
		if (res && !res._isVue && isPromise(res) && !res._handled) {
			res.catch(function (e) { return handleError(e, vm, info + " (Promise/async)"); });
			// issue #9511
			// avoid catch triggering multiple times when nested calls
			res._handled = true;
		}
	} catch (e) {
		handleError(e, vm, info);
	}
	return res
}
```

这里面除了执行原事件方法以外，还添加了 promise 错误结果处理机制，以及在嵌套调用时可能会引发的多次调用问题，这也是将原事件进行封装的原因之一。

最后就是调用 add 方法添加事件了，内部使用原生 addEventListener, 但是在这之前，会对使用微任务的环境进行一个处理，就是给事件的多次触发添加一个时间，因为异步事件会造成事件重复触发。

#### updateDirectives

常用指令有 v-text, v-html, v-if, v-for, v-else-if, v-else, v-show 等，但是这些都是内置指令，在由 AST 转换成 render 函数的时候就已经被处理，因此 updateDirectives 是专门用来处理自定义指令

```js
<div id="demo">
	<input v-focus>
</div>

directives: {
	focus: {
		inserted: function (el) {
			el.focus()
		}
	}
},
```

updateDirectives 实际上是调用 _update 方法：

```js
var isCreate = oldVnode === emptyNode;
var isDestroy = vnode === emptyNode;
var oldDirs = normalizeDirectives(oldVnode.data.directives, oldVnode.context);
var newDirs = normalizeDirectives(vnode.data.directives, vnode.context);
```

isCreate 是给 dom 初次绑定指令的标识，isDestroy 则是销毁指令标识。经过 normalizeDirectives 处理之后，newDirs 就变成如下所示

```js
newDirs = {
	'v-focus': {
		def: fn,
		modifiers: {},
		name: 'focus',
		rawName: 'v-focus'
	}
}
```

接下来就是对 newDirs 的处理，如果旧指令集中不存在该指令，将 newDirs 存入 dirsWithInsert，否则存入 dirsWithPostpatch。

```js
var dirsWithInsert = [];
var dirsWithPostpatch = [];
```

```js
if (dirsWithInsert.length) {
	var callInsert = function () {
		for (var i = 0; i < dirsWithInsert.length; i++) {
			callHook(dirsWithInsert[i], 'inserted', vnode, oldVnode);
		}
	};
	if (isCreate) {
		mergeVNodeHook(vnode, 'insert', callInsert);
	} else {
		callInsert();
	}
}
```

如果是初次给 dom 绑定指令，使用 mergeVNodeHook 进行 insert 操作。最后一个入参方法是使用 callHook 对每一个新指令进行处理。  

```js
function mergeVNodeHook (def, hookKey, hook) {
	function wrappedHook () {
		hook.apply(this, arguments);
		// important: remove merged hook to ensure it's called only once
		// and prevent memory leak
		remove(invoker.fns, wrappedHook);
	}

	if (isUndef(oldHook)) {
		// no existing hook
		invoker = createFnInvoker([wrappedHook]);
	}

	invoker.merged = true;
	def[hookKey] = invoker;
}
```

这里实际上是对 callInsert 进行了两层封装，第一层是 wrappedHook, 里面执行 callInsert 之外，还做了一个容错处理，这里就不做解析了。第二层就是通用的包装成 invoker。有一点需要注意，那就是封装完成的方法并不是赋值给 vnode['insert']，而是 vnode.data.hook['insert']。为什么呢？在 mergeVNodeHook 中有这样一段代码：def = def.data.hook || (def.data.hook = {}); 讲原因之前需要额外插一个知识点，那就是 JavaScript 中函数传值都是值传递。拿这段代码为例，在执行之前，def 指向的是 vnode，def 只是一个指针，指向 vnode；执行之后，相当于把指针指向了 {}, 那么此时的 def 就是 {}，而 vnode 并不会变成 {}，最后的结果也就变成这样 def = vnode.data.hook = {}。此时 def 和 vnode.data.hook 这两个指针指向了 {}，当为 def 添加属性时，vnode.data.hook 会同步改变，因此 def['insert'] = vnode.data.hook['insert'] = invoker, 即 vnode.data.hook['insert'] = invoker。

到这里，updateDirectives 大概就讲完了，它的主线任务就是给 vnode.data.hook 添加自定义指令对象，包括指令名和指令执行的方法。

dir.def[hook] 即上面包装两层的 invoker。

## 结语

invokeCreateHooks 执行完成后，就会调用 insert(parentElm, vnode.elm, refElm) 挂载 dom。 本文把模板到真实 DOM 的转变分成 parse、transform、generate、patch 四个步骤，前三步骤所做的事跟传统编译步骤不太一样，但也有相同点，这样划分是为了更好的在宏观层面有一个流程记忆，方便理解。每一个步骤都会涉及巨量内容及复杂情况的处理，不可能都涉及到，因此本篇的宗旨是从宏观上理解这些过程。
