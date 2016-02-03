---
layout: post
title: "带触发的lua table"
published: true
---

公司做的是MMO游戏项目，当游戏的状态发生变化，服务器会发通知客户端，客户端接到协议后，执行某些操作，给玩家提示等等。我最近在做一个模块，成就模块，就是你能想得到的那种成就，等级达到了XX给个奖励啊，收集了XX英雄给个奖励。由服务器通知客户端当前的进度。我就想这个地方完全可以做成观察者模式，于是就有了这个东西。

这是一个特殊的table，当里面的数据变化时，会回调你注册的函数。对这个可以像普通table一样赋值取值，但是不能正常遍历和应用table库函数，所以我在里面提供了代替的方法，能够完成相同的工作。测试代码：

	local function f(t, k, v)
	print(tostring(t).."     key = ".. tostring(k)..",    value = "..tostring(v) )
	end

	local t = require "triggertable"()
	local handle = t.__reg(f)  --handle can be used to unreg
	print("********test 1**********\n")
	t[1] = 1
	t[2] = {}
	print(t[1])
	print(t[2])
	print("\n********test 2**********\n")
	t[2][1] = 1
	t.__unreg(handle)          
	t[2].__reg(f)
	t[2][1] = 3
	print("\n********test 3**********\n")
	t[2].__insert(1)           --insert
	t[2].__insert(2)
	print("\n********test 4**********\n")
	t[2].__sort()              --sort
	print("\n********test 5**********\n")
	for k, v in pairs(t[2].__values) do   -- have to get __values to travel
	    print(k, v)
	end
	print("\n********test 6**********\n")
	t[2].__remove()			   --remove
	print("\n********test 7**********\n")
	print(#t[2].__values)      -- have to get __values to get the right length

###输出：

	********test 1**********

	table: 160F15C8     key = 1,    value = 1
	table: 160F15C8     key = 2,    value = table: 160F18C0
	1
	table: 160F18C0

	********test 2**********

	table: 160F15C8     key = 2,    value = table: 160F18C0
	table: 160F18C0     key = 1,    value = 3

	********test 3**********

	table: 160F18C0     key = 2,    value = 1
	table: 160F18C0     key = 3,    value = 2

	********test 4**********

	table: 160F18C0     key = 1,    value = 1
	table: 160F18C0     key = 2,    value = 2
	table: 160F18C0     key = 3,    value = 3

	********test 5**********

	1	1
	2	2
	3	3

	********test 6**********

	table: 160F18C0     key = 3,    value = nil

	********test 7**********

	2

当`t`里的数据变化，会用三个参数回调`__reg`的函数，`t`, 改变的那个键值`k`，以及`t[k]`。

###多个注册函数
对同一个表可以注册多个函数，回调的顺序由注册的顺序决定，也可以手动设置，拿到注册`__reg`时返回`handle`，执行`t.__priority(handle, level)`，level越小则回调越早，默认是5。
###把普通table变成trrigertable
假设`t`是一个lua table，那么`trrigertable(t)`之后，`t`以及`t`里面的table都会变成triggertable，每次给`t`添加新的table时也会自动转化成triggertable，如上面例子里的`t[2]`。
###好处
数据的存储与处理逻辑分离，不同模块都可以侦听同一数据的变化。
###坏处
不能像原生table那么用了，不能在处理逻辑里改变数据，一个不小心会造成循环嵌套，stack overflow。

