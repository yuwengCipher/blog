---
title: 模板怎么变成真实DOM
date: 2021-04-19
categories:
 - Vue
---

## 前言

官方有说：
> Vue.js 的核心是一个允许采用简洁的模板语法来声明式地将数据渲染进 DOM 的系统

其实 Vue 模板语法与 ejs 等模板语言类似，基本思想就是： template + data = dom。那 Vue 是如何来将模板语法结合数据转译成真实 Dom 呢？本章将会来探究这个秘密。

大致都会经历 parse、compile、generate 三个步骤，完成三个步骤就会渲染成真实的 Dom

## 编译步骤

### parse

### compile

### generate

## 结语
