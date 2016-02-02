---
layout: post
title: "Lua热更新原理(4) - 替换函数"
published: true
---

索引一个函数要么通过table，要么通过upvalue，例如全局函数放在`_G`这个表里，require的函数放在`_G.package.loaded`这个表里，我们生成一个新的函数之后，需要找到旧函数所有的索引，把这些索引指向新函数。

虚拟机里的所有值都可以通过遍历`_G`表得到，不要漏掉元表和upvalue的表，还有注意table的key也可以是函数。如果有宿主语言，那么还要遍历一下注册表，用`debug.getregistry()`获得。


