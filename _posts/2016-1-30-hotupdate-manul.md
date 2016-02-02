---
layout: post
title: Lua热更新的约定
published: true
---



本篇说明一下我的热更新的一些特性和约定。上篇在[lua热更新配置](http://asqbtcupid.github.io/hotupdte-implement/)里我们已经说到了一个重要的原则，就是只更新函数的逻辑，而不更新数据。现在我们抛开具体实现思考一下，为了让lua虚拟机把某个旧的函数替换成新的函数，我们需要提供什么？我们至少需要提供两个信息，首先旧函数是哪个，其次新函数又是哪个？那么有了约定1。

###约定1. 按文件为单位热更新
我们通过hotupdatelist指定文件名，然后热更新机制会先找之前require这个文件时产生的函数，然后重新load这个文件产生一批新的函数，用这批新的函数来替代原来找到的旧的函数。下面通过一系列例子说明什么样的函数可以被更新：

1. 局部函数`func`可以被更新
	
    	local function func()
    	end
    	return func	
        
2. 表函数`t.func`可以被更新
		
        local t ＝ {}
        function t.func()
        end
        return t
        
3. upvalue函数`f`可以被更新
	
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
    
4. upvalue表里的函数`up_t.f`可以被更新
	
        local up_t = {}
        function up_t.f()
        local function func()
            up_t.f()
        end
        return func
        
5. 元表函数`meta.f`可以被更新
		
        local meta = {}
        function meta.f()
        end
        local t = setmetatable({}, meta)
        return t
        
6. 全局函数f，或者全局表里的函数`g_t.f`可以更新
		
        function f()
        g_t = {}
        function g_t.f()
        end
        
7. 表里的表的函数`t.tt.f`可以被更新
	
    	local t ＝ {}
        t.tt = {}
        functiong t.tt.f()
        end
        return t

8. 以上规则可以自由组合，我不一一列举了

###约定2. 新增和删除函数的约定

1. 可以增加和删除upvalue，包括整个upvalue表以及upvalue函数
2. 可以增加表函数，无法删除表函数，这个“表”包括局部表，全局表，环境表，元表，upvalue表

###约定3. 全局语句执行产生未定义结果
为了避免重新加载文件时全局语句执行影响到原来的逻辑，我所采用的机制将带来一些负面影响，这些负面影响不容易描述清楚，我以后会专门写一篇文章来讨论，现在先简单声明：

1. 不会执行真正全局语句，例如：

		local t = require "somefile"
		t.somef()
   在这个例子中，不会真的`require`到`somefile`，也不会执行真的`t.somef()`，你也许想问什么叫做真	的`t.somef()`，以后再说，你就当它没执行好了
   
2. 全局语句不要有全局变量与数字的比较，例如：

		if global_a > 3 then ..... end
    
3. 通过1和2，不应该再信任全局语句的执行结果，例如:因为真的global_func()不会被执行，所以这判断的结果不会如你所想。然后这导致一个问题，比如这种写法\:

        if global_func() == true then 
            somefun = function()
                print("1")
            end
        else
            somefun = function()
                print("2")
            end
        end
   `somefun`不一定是其中的哪一个，所以就算是热更新成功，结果也会出人意料。
