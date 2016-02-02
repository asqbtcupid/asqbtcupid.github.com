---
layout: post
title: "Lua热更新原理(3) - 全局语句"
published: true
---

在[lua热更新](http://asqbtcupid.github.io/hotupdte-implement/)里提到过，重新require一个文件，会重新执行该文件的全局语句，例如：

	--example.lua
	global_var = 0
    global_func()
	local function print_some() 
    	print("something")
	end
	return print_some
   
每次清理`package.load`后，再`require "example"`都会执行`global_var = 0`和`global_func()`，这通常会破坏代码逻辑。怎么样才能不执行这些语句呢，这里提供两种思路，两种思路我都做过，各有利弊。

###第一种：语法分析
很直观的，如果我们把需要热更新的源文件读进一个字符串，然后分析这个字符串，把其中的全局语句去掉，形成一份没有全局语句的代码，那么通过`loadstring`这份没有全局语句的代码，就达到了目的。例如把example改造成：
	
    --example.lua
    local function print_some()
    	print("something")
    end
    return print_some

就是只保留函数定义的语句，


