---
layout: post
title: Lua热更新的约定
published: true
---


本篇说明一下我的热更新的一些特性和约定。上篇在[lua热更新配置](http://asqbtcupid.github.io/hotupdte-implement/)里我们已经说到了一个重要的原则，就是只更新函数的逻辑，而不更新数据。现在我们抛开具体实现思考一下，为了让lua虚拟机把某个旧的函数替换成新的函数，我们需要提供什么？我们至少需要提供两个信息，首先旧函数是哪个，其次新函数又是哪个？那么有了约定1。

###约定1. 按文件为单位热更新
我们通过hotupdatelist指定文件名，然后热更新机制会先找之前require这个文件时产生的函数，然后重新load这个文件产生一批新的函数，用这批新的函数来替代原来找到的旧的函数。下面通过一系列例子说明什么样的函数可以被更新：

1. 局部函数func可以被更新
	
    	local function func()
    	end
    	return func	
        
2. 表函数t.func可以被更新
		
        local t ＝ {}
        function t.func()
        end
        return t
        
3. upvalue函数f可以被更新
	
    可以是局部函数的upvalue：

    	local function f()
    	end
    	local function func()
    		f()
    	end
    	return func
        
    也可以是表函数的upvalue：
    
    	local t ＝ {}
        local function f()
        end
        function t.func()
        end
        return t
    
4. upvalue表里的函数up_t.f可以被更新
	
        local up_t = {}
        function up_t.f()
        local function func()
            up_t.f()
        end
        return func
        
5. 元表函数meta.f可以被更新
		
        local meta = {}
        function meta.f()
        end
        local t = setmetatable({}, meta)
        return t
        
6. 全局函数f，或者全局表里的函数g_t.f可以更新
		
        function f()
        g_t = {}
        function g_t.f()
        end
        
7. 表里的表的函数t.tt.f可以被更新
	
    	local t ＝ {}
        t.tt = {}
        functiong t.tt.f()
        end
        return t

###约定2. 新增和删除函数的约定

1. 可以增加和删除upvalue，包括整个upvalue表以及upvalue函数
2. 可以增加表函数，无法删除表函数，这个“表”包括局部表，全局表，环境表，元表，upvalue表

###约定2. 全局语句执行产生未定义结果




 

