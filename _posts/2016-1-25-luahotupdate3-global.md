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
   
每次`require "example"`都会执行`global_var = 0`和`global_func()`，这显然不能