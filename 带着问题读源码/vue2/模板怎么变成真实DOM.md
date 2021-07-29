---
title: 模板怎么变成真实DOM
date: 2021-04-19
categories:
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
3. var compileToFunctions = ref$1.compileToFunctions;
4. $mount();
5. var res = compileToFunctions();
6. var compiled = compile(template, options)
7. compiled = baseCompile(template.trim(), finalOptions)
8. compiled.ast = parse(template.trim(), options)
9. compiled.render = generate(ast, options).render
10. res.render = createFunction(compiled.render, fnGenErrors);

## 转译步骤

### parse

这一步将 template 作为字符串进行特定规则分析转换成 AST。

### transform

这一步通过 AST 生成 render 函数方法

### generate

这一步调用 render 函数生成 vnode

## patch

## 结语
