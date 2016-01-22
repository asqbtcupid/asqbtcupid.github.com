---
layout: post
title: Lua closure 热更新
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

也就是当require过一次example以后，即使example的内容变了，但是package.loaded["example.lua"]已经不是nil，所以新的exmaple并不会被执行，如果想要require到新的example，那么只需要先package.loaded["example.lua"]，然后再require "exmaple"，就能执行到新的exmaple了



