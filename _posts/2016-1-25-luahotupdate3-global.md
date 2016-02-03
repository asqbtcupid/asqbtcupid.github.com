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
   
每次清理`package.load`后，再`require "example"`都会执行`global_var = 0`和`global_func()`，这通常会破坏代码逻辑。怎么样才能不执行这些语句呢，这里提供两种思路，两种思路我都做过，各有利弊。

##第一种：语法分析
很直观的，我们把需要热更新的源文件读进一个字符串，然后分析这个字符串，只保留函数定义的语句，把多余的全局语句(调用函数的、设置全局变量的)去掉，之后`loadstring`这份代码，就达到了目的。例如把example变成了：
	
    --example.lua
    local function print_some()
    	print("something")
    end
    return print_some

###需要为upvalue占坑
例如有文件是这样的：

	--example.lua
	local i = get_a_number()   -- get_a_number是全局函数，返回一个数字
	local function func()
		print(i)
	end
	return func

在[lua热更新原理(2) - upvalue](http://asqbtcupid.github.io/luahotupdate2-upvalue/)有说，我们需要把旧函数的upvalue复制到新函数里，在lua5.1里，只能改变upvalue的值，而不能新添upvalue。对于上面这个例子，语法分析时，不能把`local i = get_a_number()`整句去掉。因为去掉之后，`func`里就没有了`i`这个upvalue，也就没办法把旧函数的`i`复制过来。

在这里，我们可以把`local i = get_a_number()`变成`local i = {}`或者别的什么，保留住这个`i`就行。

###怎么语法分析？
[lua 5.1的文法](http://www.lua.org/manual/5.1/manual.html#8)，自己写一个完备的语法分析器，或者用工具生成。

或者像我偷工减料，只是用lua的string库分析分析，只能识别几种常用的函数定义，当时对我们的项目来说，绝大部分lua文件都OK，我也就懒得写语法分析了。我当时写的代码如下，输入一个lua文件的系统路径，然后`print`改造后的源码。

	function BuildNewCode(FilePath)
		io.input(FilePath)  
		local chunk = ""
		local LocalVar = {}
		local GlobalVar = {}
		local FunctionDeepth = 0
		local needend = 0
		local ReturnExist = false
		local IsInComment = false
		local FunctionType = {}  -- 0:global 1:local 2:table,3. anonymous
		local IsFirstLine = true
		for line in io.lines() do
	    local OriginalLine = line
	    line = string.gsub(line, "\\.", "")
	    line = string.gsub(line, "\".-\"", "")
	    if IsFirstLine then
	      IsFirstLine = false         
	    else 
	      chunk = chunk.."\n"
	    end
	    if ReturnExist then chunk = chunk..line end
	    line = string.gsub(line, "%-%-[^%[%]].*", "")
	    if string.find(line, "--%[%[") then
	      IsInComment = true
	    end
	    if IsInComment and string.find(line, "%]%]") then
	      IsInComment = false
	      line = string.gsub(line, "%-%-.*", "")
	      line = string.gsub(line, ".*%]%]", "")
	      OriginalLine = line
	    end
	    if not IsInComment then 
	      if string.find(line, "^function$") or string.find(line, "[^%w]+function$") or string.find(line, "^function[^%w]+") or string.find(line,"[^%w]+function[^%w]+") then 
	          local funtype = -1
	          if string.find(line, "^local%s+.*=[^%w]*function") or string.find(line, "%s+local%s+.*=[^%w]*function") then 
	            funtype = 3
	          elseif string.find(line, "local%s+function") then 
	            funtype = 1
	          end
	          FunctionDeepth = FunctionDeepth + 1 
	        needend = needend + 1
	          if funtype ~= 1 and funtype ~= 3 then
	            local head, body, point, tail = string.match(OriginalLine, "(.*function%s+)([%w_]+)([:.])([%w_]+%(.*)")
	            if point ~= nil and point ~= "" then
	                local varnametemp = string.match(body,"[%w_]+")
	                if LocalVar[varnametemp] then
	                  funtype = 2
	                elseif GlobalVar[varnametemp] == true then
	                  LocalVar[varnametemp] = true
	                  chunk = chunk.."local "..varnametemp.." = {}"
	                  funtype = 2
	                else
	                  funtype = 0
	                end
	            elseif FunctionDeepth == 1 then
	                funtype = 0
	            else
	                funtype = 1
	            end
	          end
	          table.insert(FunctionType, funtype)
	      end
	      if string.find(line, "^if$") or string.find(line, "[^%w_]+if$") or string.find(line, "^if[^%w_]+") or string.find(line,"[^%w_]+if[^%w_]+") then
	        needend = needend + 1
	      end     
	      if string.find(line, "^do$") or string.find(line, "[^%w_]+do$") or string.find(line, "^do[^%w_]+") or string.find(line,"[^%w_]+do[^%w_]+") then
	          needend = needend + 1
	      end
	      if FunctionDeepth == 0 and (string.find(line, "^return%s+") or string.find(line, "%s+return%s+")) then
	          chunk = chunk..OriginalLine
	          ReturnExist = true
	      end

	      if FunctionDeepth == 0 then
	          local varnames = string.match(line, "^local%s+([^=]+)=?") or string.match(line, "%s+local%s+([^=]+)=?") 
	          if varnames ~= nil  then
	            varnames = string.gmatch(varnames, "[_%w]+")
	            if varnames ~= nil then
	                for name in varnames do
	                  OriginalLine = "local "..name.." = {}"
	                  LocalVar[name] = true
	                  chunk = chunk..OriginalLine
	                end
	            end
	        else
	            varnames = string.match(line, "^([%w_]+)%s*=") or string.match(line, "%s+([%w_]+)%s*=")
	            if varnames ~= nil then
	              GlobalVar[varnames] = true
	            end
	          end
	      end

	      if FunctionDeepth > 0 and ( FunctionType[1] == 2 or FunctionType[1] == 1 or FunctionType[1] == 3 ) then
	          chunk = chunk..OriginalLine
	      end
	      if string.find(line, "^end$") or string.find(line, "[^_%w]+end$") or string.find(line, "^end[^_%w]+") or string.find(line,"[^_%w]+end[^_%w]+") then
	          needend = needend - 1
	          if needend < FunctionDeepth then
	            FunctionDeepth = FunctionDeepth - 1
	            FunctionType[#FunctionType] = nil
	          end
	      end
	    end
	  end
	  io.input():close()
	  print(chunk)
	end

##第二种：用假的环境表

这是现在的版本在用的，思路跟语法分析截然相反，不改造源文件，全局语句让它执行，但是执行之后没有效果就好。首先请温习一下[元表的用法](http://www.lua.org/manual/5.1/manual.html#2.8)。

###环境表
lua里的每个函数都绑定了一个表，叫做环境表。在一个函数里索引一个变量，首先从局部变量找，其次从upvalue找，最后会去环境表里找。

默认情况下全局变量都放在`_G`这个表中，例如下面的代码：

	i = 6
	local function f()
		print(i)
		_G["print"](_G["i"])
	end

`print(i)`与`_G[print](_G["i"])`是一样的，我们通过改变环境表来做手脚。

我们先把文件源代码读进字符串，通过`loadstring`源代码字符串可以得到一个函数，执行该函数就能得到代码最后的那个`return`的东西（如果有`return`）。但是在执行之前，我们用`setfenv`来改变这个函数的环境表。那改成什么呢？
###假环境表




