---
layout: post
title: "Lua Closure 热更新(2) - Upvalue"
published: true
---
##对table的误解

对于lua新手来说，可能会把table当成了c++里的class或者struct，就像当初的我，例如：
	--example.lua
	local t = {}
	t.data = 6
	function t.func()
		print(t.data)
	end
	return t

可能你会觉得`t.func`能访问`t.data`是因为它们都是`t`里的值，当你把`t`认为是一个class，自然会认为`t.func`具有访问“数据成员”`t.data`的权限。你是不是觉得无论何时何地，table里的函数应该总能访问到该table里的其它“成员”？那如果有两个文件是这么写的：

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
我想用一句话来总结：**函数里用到的定义在该函数之前的local变量，就成为了该函数的upvalue**。在此我还要说一个关于local变量误区，也是因为把它类比成了其它语言的局部变量造成的，就是误解了它的生命周期。在lua里，对于local变量，只要你还能访问到，不管是通过upvalue还是通过table，那么这个local变量就不消失，它的值也不会变化除非你主动改变它。对于这个文件：
	--example.lua
	local t = {}
	t.data = 6
	function t.func()
		print(t.data)
	end
	return t

`t.func`能访问到`t.data`，是因为`t`成为了`t.func`的upvalue。

##热更新需要注意的
把旧函数换成新函数，也要记得把旧函数的upvalue复制新函数身上。例如:

	--example.lua
	local count = 0
	local function func()
		count = count + 1
		print(count)
	end
	return func

`count`用来统计`func`运行了多少次，如果你用[Lua热更新原理(1)](http://asqbtcupid.github.io/luahotupdate1-require/)的方法来热更新，那么需要把原来函数的`count`复制过来。可以使用`debug.getupvalue`和`debug.setupvalue`来完成，简单的代码如下：

	local oldfunc = require "example"
	package.loaded["example"] = nil
	local newfunc = require "example"

	for i = 1, math.huge do
		local name, value = debug.getupvalue(oldfunc, i)
		if not name then break end
		debug.setupvalue(newfunc, i, value)
	end

通过这种方法，就能把旧函数的upvalue复制到新函数里。



