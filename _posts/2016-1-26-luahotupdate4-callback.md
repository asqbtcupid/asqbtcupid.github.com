---
layout: post
title: "Lua热更新原理(4) - 替换函数"
published: true
---
现在我们得到了新的函数，也知道旧的函数是什么，接下来要做的就是遍历虚拟机，
找到旧函数所有的索引，并把这些索引指向新函数。

虚拟机里的所有值都可以通过遍历`_G`取得，需要注意的是：

1. 不要漏掉元表和upvalue的表，元表的获取用`debug.getmetatable`，这样对于有`metatable`这个key的元表，也能正确获取。
2. 还有注意table的key也可以是函数。
3. 如果有宿主语言，那么还要遍历一下注册表，用`debug.getregistry()`获得。

下面这个函数可以做这件事情：

	function replace(oldfunction, newfunction)
		local visited = {}
		local function f(t)
			if not t or visited[t] then return end
			visited[t] = true
			if type(t) == "function" then
			  	for i = 1, math.huge do
					local name, value = debug.getupvalue(t, i)
					if not name then break end
					f(value)
				end
			elseif type(t) == "table" then
				f(debug.getmetatable(t))
				for k,v in pairs(t) do
					f(k); f(v);
					if type(v) == "function" or type(k) == "function" then
						if v == oldfunction then t[k] = newfunction end
						if k == oldfunction then 
							t[newfunction] = t[k]
							t[k] = nil
						end
					end
				end
			end
		end
		f(_G)
		local registryTable = debug.getregistry()
		for k, v in pairs(registryTable) do
			if v == oldfunction then
				registryTable[k] = newfunction
			end
		end
	end

##至此关于lua热更新原理就介绍完毕了
