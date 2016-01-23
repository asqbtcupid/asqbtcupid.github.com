---
layout: post
title: "Lua Closure 热更新(1) - require机制"
published: true
---



### require 做了什么？
把下面的代码写到一个lua文件里，比如叫example.lua

    local function print_some()
    	print("something")
    end
    return print_some

那么`require "example.lua"`可以用下面的语句实现

    if package.loaded["example.lua"] == nil then	--package是默认就有的全局table
        function f()
        	local function print_some()			  --注意只有这四行是
           		print("something") 				--example.lua的内容
           end
           return print_some                   
        end
        local result = f()
        package.loaded["example.lua"] = result or true
    end
    return package.loaded["example.lua"]
    
根据这个语义，多次执行require "exmaple.lua"，也只有第一次执行exmaple里的内容，执行之后的返回值会缓存到package.loaded里，再次require就不会执行example，而是直接返回package.loaded["example"]。

如果你改写了example的内容，然后想require改写之后的example，那怎么办呢？只需要先执行package.loaded["example.lua"] = nil，那么再require "exmaple"时，就会得到新的example。如下面的代码
	
	package.loaded["example.lua"] = nil
	local func = require "example"
   
每次调用这两句，func都会是新的，这相当于实现了热更新，别高兴，这只适用于非常简单的情况。工程上没法这么用，别的不说，首先每次都会require都要调用package.loaded["example.lua"] = nil，这需要修改逻辑代码。我们想要的是一种非侵入的方式实现热更新

其次因为example的内容会重新执行一次，会重新执行你不想要执行的代码，假如example是这么写的
	global_var = 0
	local function print_some()	
    	print("something")
	end
	return print_some
那么每一次“热更新”都会把全局变量global_var置为0，到时候程序叮铛出错的时候你还不知道怎么回事~
