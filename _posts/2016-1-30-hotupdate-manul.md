---
layout: post
title: Lua热更新的约定
published: true
---


本篇说明一下我的热更新的一些特性和约定。上篇在[lua热更新配置](http://asqbtcupid.github.io/hotupdte-implement/)里我们已经说到了一个重要的原则，就是只更新函数的逻辑，而不更新数据。现在我们抛开具体实现思考一下，为了让lua虚拟机把某个旧的函数替换成新的函数，我们需要提供什么？我们至少需要提供两个信息，首先旧函数是哪个，其次新函数又是哪个？那么有了约定1。

###约定1. 按文件为单位热更新
我们通过hotupdatelist指定文件名，然后热更新机制会先找之前require这个文件时产生的函数，然后重新load这个文件产生一批新的函数，用这批新的函数来替代原来找到的旧的函数。通过一系列例子说明(文件中出现的函数都可以被更新)：
1. 局部函数可以被更新
	--example1.lua	
    local function func()
    end
    return exmaple
2. 局部函数所引用的局部函数f可以被更新
	--example2.lua
    local function f()
    end
    local function func()
    	f()
    end
    return exmaple
3. item


	


###约定2. 全局语句执行产生未定义结果




 

