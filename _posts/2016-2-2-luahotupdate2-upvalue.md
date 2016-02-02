---
layout: post
title: "Lua Closure 热更新(2) - Upvalue"
published: true
---
##对table的误解

对于lua新手来说，可能会把table当成了c++里的class或者struct，就像当初的我，例如：

	local t = {}
	t.data = 6
	function t.func()
		print(t.data)
	end
	return t

可能你会觉得`t.func`能访问`t.data`是因为它们都是t里的值，当你把`t'认为是一个class，自然会认为`t.func`具有访问“数据成员”`t.data`的权限。你是不是觉得无论何时何地，table里的函数应该总能访问到该table里的其它“成员”？那如果有两个文件是这么写的：

	--t_func.lua
	local function func()
		print(t.data)
	end
	return func

   和

	--t.lua
	local t = {}
	t.data = 6
	t.func = require "t_func"
	return t

当你调用`t.func()`，还会得到想要的结果吗？不会，你会得到一个报错`attempt to index global 't' (a nil value)`，这时候的`t.func`压根不知道`t`的存在，更别说't.data'了。

table对于该table里的值并不一定可见，upvalue是维系它们的桥梁。

##什么是upvalue？
我想用一句话来总结：**函数里用到的定义在该函数之前的local变量，就成为了该函数的upvalue**。
