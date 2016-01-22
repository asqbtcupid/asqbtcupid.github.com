---
layout: post
title: Lua closure 热更新
published: true
---


require 做了什么？

一个lua文件example.lua，内容如下

    local i = 3
    return i

那么require "example.lua"可以用下面的语句实现

    if package.loaded["example.lua"] == nil then
    
        function f()
    
            local i = 3    --注意只有这两句是
    
            return i       --example.lua的内容
    
        end
    
        local result = f()
    
        package.loaded["example.lua"] = result or true
    
    end
    
    return package.loaded["example.lua"]

根据这个语义，多次执行require "exmaple.lua"，也只有第一是真正的执行exmaple里的内容。这时如果在require之前把package.loaded["example.lua"] = nil，那么当再require "exmaple"，就会再执行一次exmaple的内容了，料你已经想到，这时候如果exmaple的内容变了，那不就是相当于热更新了吗。但是问题随之而来。
