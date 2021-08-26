---
title: 简单说说 Vue diff 算法
date: 2021-06-25
index_img: 'https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/thumbnail/简单说说Vue-diff算法.jpg'
categories:
 - Vue
tags:
 - Vue
---

## 前言

diff 针对的目标是 vnode 即 virtual dom。那为什么需要 vnode 呢？根本目的不是为了节约 DOM 操作的性能开销，而是可以多端使用。如果根标签下只有一个文本，我直接使用原生方法去操作 DOM 肯定是要比使用 vnode 更好的，因此说 vnode 节约性能开销还是会需要基于内容多少去谈。vnode 是字符串模板经过 AST 转换，再到 render 函数转换，最后由 render 函数生成的，那么就可以用于服务端，由服务端去创建 html 并返回给客服端使用。

在[模板怎么变成真实DOM](https://djacipher.cn/2021/04/19/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/vue2/%E6%A8%A1%E6%9D%BF%E6%80%8E%E4%B9%88%E5%8F%98%E6%88%90%E7%9C%9F%E5%AE%9EDOM/)篇中，我们将侧重点放在了整个渲染流程，关注的是初始渲染。那这一篇就来看看当页面更新渲染时发生的 patchVnode，以及涉及到的 diff 算法。

修改 demo 如下：

```js
// html
<div id="demo">
	<ul>
		<li v-for="(branch, index) in branches">{{branch}}</li>
	</ul>
</div>

// js
data: {
	branches: ['A', 'B', 'C', 'D'],
},
mounted () {
	setTimeout(() => {
		this.branches = ['A', 'E', 'B', 'F', 'C', 'D']
	}, 1000)
},
```

## patchVnode

跟初次渲染走 createElm 分支不同的是，更新渲染执行的是 patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)

```js
function patchVnode (oldVnode, vnode, insertedVnodeQueue, ownerArray, index, removeOnly) {
	// 省略
	var oldCh = oldVnode.children;
	var ch = vnode.children;
	if (isUndef(vnode.text)) {
		if (isDef(oldCh) && isDef(ch)) {
			if (oldCh !== ch) { updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly); }
		} else if (isDef(ch)) {
			{
				checkDuplicateKeys(ch);
			}
			if (isDef(oldVnode.text)) { nodeOps.setTextContent(elm, ''); }
			addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
		} else if (isDef(oldCh)) {
			removeVnodes(oldCh, 0, oldCh.length - 1);
		} else if (isDef(oldVnode.text)) {
			nodeOps.setTextContent(elm, '');
		}
	} else if (oldVnode.text !== vnode.text) {
		nodeOps.setTextContent(elm, vnode.text);
	}
}
```

我们省略掉不重要的处理之后，剩下的就是需要关注的重点。

依据新 vnode.text 是否存在分为两种情况进行处理。如果存在 vnode.text 意味着当前 vnode 就是文本类型，那么只需在判断 oldVnode 与 vnode 文本不相等之后重新更新 text 就行。如果 vnode.text 不存在又分为多种情况。

第一种：oldCh 与 ch 都存在并且它们不相等，调用 updateChildren 处理 oldCh 与 ch。
第二种：只存在 ch，说明是新增加的 child。
- 首先检查 child 的 children 的 key 是否重复
- 如果 oldVnode.text 存在，则需将 elm 的文本内容赋值为空字符串
- 最后调用 addVnodes 将 ch 添加到 elm 中
第三种：只存在 oldCh, 说明该 oldCh 需要被删除掉，所以调用 removeVnodes 执行删除操作
第四种：oldCh 与 ch 都不存在，但是 oldVnode.text 存在，那么也是会将 elm 的文本内容赋值为空字符串

接下来我们来看看 updateChildren 是如何处理 children 的。

## diff

当 newVnode 拥有 children 并且当 oldCh 和 ch 不相等时，这时就会执行 updateChildren 对 oldCh 和 ch 进行 diff 操作。

```js
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
	var oldStartIdx = 0;
	var newStartIdx = 0;
	var oldEndIdx = oldCh.length - 1;
	var oldStartVnode = oldCh[0];
	var oldEndVnode = oldCh[oldEndIdx];
	var newEndIdx = newCh.length - 1;
	var newStartVnode = newCh[0];
	var newEndVnode = newCh[newEndIdx];
	var oldKeyToIdx, idxInOld, vnodeToMove, refElm;
}
```

这部分是 diff 操作的准备工作。对于新旧 vnode，分别有各自的 startIdx、endIdx、startVnode、endVnode。

然后就开始真正的 diff 过程。通过 while 循环来按照一定的规则比对新旧 vnode，最后执行新增、更新或者移除操作。比对的顺序是从两端向中间移动，这些规则分为7种情况。

另外为了表述的简洁，我在这里将 startIdx 向右移动一位并且 startVnode 重新指向这个 vnode 称为 startVnode 右进一位，将 endIdx 向左移动一位并且 endVnode 重新指向这个 vnode 称为 endVnode 左进一位。

### diff 情形

#### 第一种

```js
if (isUndef(oldStartVnode)) {
	oldStartVnode = oldCh[++oldStartIdx];
}
```

如果 oldStartVnode 不存在，oldStartVnode 右进一位。

#### 第二种

```js
if (isUndef(oldEndVnode)) {
	oldEndVnode = oldCh[--oldEndIdx];
}
```

如果 oldEndVnode 不存在，oldEndVnode 左进一位。

#### 第三种

```js
if (sameVnode(oldStartVnode, newStartVnode)) {
	patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
	oldStartVnode = oldCh[++oldStartIdx];
	newStartVnode = newCh[++newStartIdx];
}
```

进入这一分支，说明 oldStartVnode 和 newStartVnode 都存在，那么就会比较他们是否是同一个 vnode。那什么情况是 sameVnode 呢？看看它是如何判断的吧。

```js
function sameVnode (a, b) {
	return (
		a.key === b.key &&
		a.asyncFactory === b.asyncFactory && (
			(
				a.tag === b.tag &&
				a.isComment === b.isComment &&
				isDef(a.data) === isDef(b.data) &&
				sameInputType(a, b)
			) || (
				isTrue(a.isAsyncPlaceholder) &&
				isUndef(b.asyncFactory.error)
			)
		)
	)
}
```

可以里面涉及到许多属性的判断，在这里就不具体去说了。为了便于理解，依据我们当前 demo 来说，就是判断 key 是否相等。

由于我们没有设置 key 属性，因此这里的以及后面的 sameVnode 判断都默认为 true。

判断通过后，就调用 patchVnode 处理 oldStartVnode 和 newStartVnode。处理完成后，oldStartVnode 和 newStartVnode 都右进一位。

#### 第四种

```js
if (sameVnode(oldEndVnode, newEndVnode)) {
	patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx);
	oldEndVnode = oldCh[--oldEndIdx];
	newEndVnode = newCh[--newEndIdx];
}
```

如果 oldEndVnode 和 newEndVnode 是同一个 vnode，同样使用 patchVnode 处理它们，完成后 oldEndVnode 和 newEndVnode 都左进一位。

#### 第五种

```js
if (sameVnode(oldStartVnode, newEndVnode)) {
	patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx);
	canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm));
	oldStartVnode = oldCh[++oldStartIdx];
	newEndVnode = newCh[--newEndIdx];
}
```

进入这一分支，说明 oldStartVnode 与 newStartVnode 不相等并且 oldEndVnode 与 newEndVnode 也不相等。那么此时需要将 oldStartVnode 与 newEndVnode 进行比较，这样比较的原因是考虑到一种情况，那就是原来旧 vnode 中的某一项在新 node 树中位置向右移动了，那么就不用进行移除和新增 dom，而只需要移动 dom 就行，节约了移除和新增的新能开销。

处理完成后，oldStartVnode 向右进一位，newEndVnode 向左进一位。

#### 第六种

```js
if (sameVnode(oldEndVnode, newStartVnode)) {
	patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
	canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
	oldEndVnode = oldCh[--oldEndIdx];
	newStartVnode = newCh[++newStartIdx];
}
```

这一种与上面说的情况相反，考虑的是原来旧 vnode 中的某一项在新 node 树中位置向左移动了，因此处理 oldEndVnode 和 newStartVnode。

处理完成后，newStartVnode 向右进一位，oldEndVnode 向左进一位。

#### 第七种

进入这个分支，说明新旧 vnode 树前后都不相等。我们来看看它是如何处理这种情况的

```js
if (isUndef(oldKeyToIdx)) { oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx); }
idxInOld = isDef(newStartVnode.key)
	? oldKeyToIdx[newStartVnode.key]
	: findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);
```

oldKeyToIdx 是未处理 oldVnode 在 oldCh 中的位置信息，比如说现在就剩 B 没有处理，并且都设置了 key 为自身，如下所示

```html
<li v-for="(branch, index) in branches" :key="branch">{{branch}}</li>
```

那么 oldKeyToIdx 就像这样

```js
oldKeyToIdx = {
	B: 1
}
```

对象集合中的 key 是 Vnode.key，value 是在 oldCh 中的下标。

然后就是获取 newStartVnode 在 oldCh 里的位置下标，分为两种情况。一是如果 newStartVnode.key 存在，那么就拿这个 key 去 oldKeyToIdx 匹配；二是 newStartVnode.key 不存在，调用 findIdxInOld 去寻找下标。

```js
if (isUndef(idxInOld)) {
	createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx);
} else {
	vnodeToMove = oldCh[idxInOld];
	if (sameVnode(vnodeToMove, newStartVnode)) {
		patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
		oldCh[idxInOld] = undefined;
		canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm);
	} else {
		createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx);
	}
}
```

到这里就可以拿到 idxInOld 了。接下来就分两种情况进行操作了：
- 如果 idxInOld 不存在，说明 newStartVnode 是新增的，那么就调用 createElm 创建新 DOM 并插入 DOM；
- 如果 idxInOld 存在，就在 oldCh 中找到这个下标的 vnode，赋值给 vnodeToMove。
  - 如果 vnodeToMove 与 newStartVnode 相等，则调用 patchVnode 处理它们。处理完成后，将 oldCh[idxInOld] 赋值为 undefined，然后执行移动操作。
  - 如果不相等，说明它们的 key 相同，但由于是不同的 elm，那么也还是按照新增处理，调用 createElm 方法。

上面的处理完成后，newStartVnode 向右进一位。

#### 新增或是移除

两棵 vnode tree 如下：

![patch_tree](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/patch_tree.png)

理想状态是经过上述7种情况的处理，如果新旧 vnode 的数量是一样的，那么 oldCh 和 ch 都会同时处理完成。但是按照文章开头设置的 dmeo 来执行的话，最后会剩下 C、D 两个节点没有处理，那么就会执行新增操作。如果将例子相反设置，那么 E、F 就会被执行移除操作。

相信你发现了一个问题，那就是处理 C、D 的新增操作其实是不必要的，因为它们本来就在 oldCh 中存在，我们只需要复用就行了。那是什么原因导致没有按照设想的步骤来执行的呢？

### diff 过程

在上面的 demo 中，我们在用 v-for 渲染列表时并没有添加 key 属性。也就是因为没有设置 key，在比较 sameVnode 时 由于 undefined === undefined，sameVnode() 一直为 true，所以才会一直执行第三种情况，自始至终都是都是从头向尾进行 patch。

由于 oldCh 长度大于 ch，当 ch 处理完之后，oldCh 还剩两项，进入下一轮比较时，oldStartIndx(4) > oldEndIndx(3)，就会跳出循环，执行 addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)。其中 newStartIdx = 3，newEndIdx = 5，它们的中间就是 C 和 D 所对应的 li vnode，那么这两个 vnode 就会被拿去创建成 DOM。

我们都知道官方提醒我们在使用 v-for 渲染列表时要给每一项都设置独一无二的 key，这是为什么呢？因为独一无二的 key 可以匹配真正相同的 vnode，过滤假的相同。这句话怎么理解呢？

我们先给每一项都设置 key 为 index，如下：

```html
<li v-for="(branch, index) in branches" :key="index">{{branch}}</li>
```

按理说一个列表中的 index 是不可能相等的，但是如果按照 sameVnode 的判断逻辑来说，B 所对应的 li 与 E 所对应的 li 依然还是相同的，因为它们所处的位置下标都是1，那么 key 都是 1，因此 key 设置成 index 是无效的。

如果我们把 key 设置成 branch，B 和 E 是不可能相等的，那么就过滤掉了这个假相同，达到了我们的目的。因此我们平时在写业务代码时可以把 key 设置成 id。

```html
<li v-for="(branch, index) in branches" :key="branch">{{branch}}</li>
```

按照前后顺序进行处理，当新旧树中相同部分的 A、C、D 被处理结束后，就剩如下所示：

![步骤](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/diff_1.png)

此时就会进入第七种情况处理 E。去获取 E 在 oldCh 中的下标，因为 oldCh 不存在 E，所以 idxInOld = undefined，那么就当做新 elm，做新增处理，它在 ch 中的位置下标为 1，那么就会插入到下标为1的位置。

然后进入第三种情况处理 B。处理完之后只剩下 F，又会被当做新 elm，做新增处理，它在 ch 中的位置下标为 3，插入到下标为3的位置。

## 最后

这篇文章从源码的角度简单讲述了 Vue diff 的过程，目的是对 diff 原理有一个大框架上的理解。简单总结一下：

1. diff 采用的策略是从两头往中间进行 diff patch
2. 新旧树中存在多个相同 vnode，只是位置不同时，设置独一无二的 key 有助于减少创建 DOM 带来的性能消耗
