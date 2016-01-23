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

    if package.loaded["example.lua"] == nil then   --package是默认就有的全局table
        function f()
        	local function print_some(）       --注意只有这四行是
           		print("something") 				--example.lua的内容
           end
           return print_some                   
        end
        local result = f()
        package.loaded["example.lua"] = result or true
    end
    return package.loaded["example.lua"]
    
根据这个语义，多次执行require "exmaple.lua"，也只有第一是真正的执行exmaple里的内容。这时如果在require把package.loaded["example.lua"] = nil，那么当再require "exmaple"，就会再执行一次exmaple的内容了，料你已经想到，这时候如果exmaple的内容变了，那不就是相当于热更新了吗。但是问题随之而来。
