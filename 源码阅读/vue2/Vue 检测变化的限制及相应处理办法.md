---
title: Vue 检测变化的限制及相应处理办法
date: 2021-06-07
index_img: 'https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/thumbnail/Vue检测变化的限制及相应处理办法.jpg'
categories:
 - Vue
tags:
 - Vue
---

## 前言

官方文档说：由于 JavaScript 的限制，Vue 不能检测数组和对象的变化。然后分别列出了对象及数组中存在的限制情况和解决办法。由于它只说明是 JavaScript 的限制，具体原因没有提及，这难免会让人感到些许疑惑。因此本篇就是专门记录这个限制的原因及解决办法的原理。

## 限制的真正原因

大家都知道，Vue 使用 Object.defineProperty 去对 data 中的数据增加数据劫持，然后这些数据就拥有了响应式属性。那会是 Object.defineProperty 的限制吗？先看数组方面的限制，我们写一个小 demo 测试一下：

```js
let arr = [1, 2]
Object.defineProperty(arr, 0, {
    get: function() {
        return 1
    },
    set: function(newVal) {
        console.log('set value: ' + newVal)
    }
})
arr[0] = 3
```

将数组下标 0 位置做了数据劫持，然后执行 arr[0] = 3，发现控制台打印除了 set value: 3。也就是说对数组做数据劫持是没有问题的。那 Vue 是怎么处理数组的呢？我们来看下官方 demo，修改如下：

```js
// html
<div id="demo">
	{{ JSON.stringify(branches) }}
</div>

// js
data: {
	branches: ['master', 'dev', { testArray: 'tag_0' }],
}
mounted () {
	setTimeout(() => {
		this.branches[0] = 'tag_1'
	}, 1000)
},
```

1秒之后将 branches.master 赋值为 'tag_1', 但是页面依然显示的是 ["master","dev",{"testArray":"tag_0"}]，跟官方描述的一样，这样的设置是不会更新页面的。

我们找到源码中的 initData 方法。它最终会执行 observe(data, true /* asRootData */)。

```js
function observe (value, asRootData) {
	if (!isObject(value) || value instanceof VNode) {
		return
	}
	var ob;
	if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
		ob = value.__ob__;
	} else if (
		shouldObserve &&
		!isServerRendering() &&
		(Array.isArray(value) || isPlainObject(value)) &&
		Object.isExtensible(value) &&
		!value._isVue
	) {
		ob = new Observer(value);
	}
	if (asRootData && ob) {
		ob.vmCount++;
	}
	return ob
}
```

想要真正的拥有响应式属性，必须得执行 ob = new Observer(value) 这一段代码，但是由于我们所修改的第一项 'master' 并不是对象，因此不会执行到下面，也就是说 master 属性不是响应式的，那么当修改它时，Vue 就无法检测了。

那只要是数组里的属性都无法通过下标修改吗？我们试着修改一下数组中第三项的属性:

```js
mounted () {
	setTimeout(() => {
		this.branches[2].testArray = 'tag_1'
	}, 1000)
},
```

可以看到页面在1秒之后显示为 ["master","dev",{"testArray":"tag_1"}]。说明 Vue 是可以监测到数组属性的修改。因为 this.branches[2] 是一个对象，通过了第一层判断，会执行添加响应式属性的代码，所以 Vue 能检测到变动。

总结起来，原因是 Vue 对数组中的非对象属性不会添加响应式属性，所以通过下标修改的属性如果不是对象，Vue 是检测不到的，而如果是对象则能正常响应。

至于为什么这样设计，尤大大解释是对性能及用户体验性价比的考量。

那对于对象而言，假如现在根对象有一个对象 anotherBranch: {}, 当我们使用 this.anotherBranch.tag = 'tag_3'，那么这个 tag 属性也不是响应式的。了解生命周期的应该知道，当我们添加属性时，添加响应式属性过程已经执行完毕了。

## 如何回避限制

有了这样的设计，官方也给出了如何去回避这些限制的解决办法。对象和数组的解决办法中有一个相同点，那就是使用 $set 方法。比如说我想修改 master 的值，那我可以这样用：

> this.$set(this.branches, 0, 'tag_1')

1秒之后页面显示的是  ["tag_1","dev",{"testArray":"tag_0"}]，符合预期！那接下来我们就来看看这个方法的奥秘。

```js
Vue.prototype.$set = set;

function set (target, key, val) {
	// 省略
	if (Array.isArray(target) && isValidArrayIndex(key)) {
		target.length = Math.max(target.length, key);
		target.splice(key, 1, val);
		return val
	}
	// 省略
	defineReactive(ob.value, key, val);
	ob.dep.notify();
	return val
}
```

对于数组，就是执行 Array.splice() 方法修改了数组的值，也就是修改了 currentBranch 的值。由于 currentBranch 是 data 对象中的一个属性，拥有响应式属性，现在发生了改变，就会触发它的 set 方法，那么就会触发页面的重新渲染。而在这次渲染之前，master 变成了 tag_1，所以页面中 master 变成了 tag_1。其实不仅仅是 splice 方法可以实现，只要是能修改数组的方法就可以回避这个限制。这个回避策略也可以理解成将数组属性的变化转给数组的变化。

对于对象，就是在我们手动添加属性后，重新执行 Object.defineProperty 为该属性增加响应式属性，并执行 dep.notify() 重新收集依赖，然后就可以在后期检测到变化并且能通知订阅者更新。

## 最后

总结起来，Vue 检测变化的限制实际上是这些变动的属性没有响应式属性。其中，数组属性是官方基于性能方面的考虑没有添加响应式属性，而对象是因为错过了添加响应是属性的时机。对应的解决办法就是：数组属性的变化转接给数组的变化、对象属性重新添加响应式属性并重新收集依赖。
