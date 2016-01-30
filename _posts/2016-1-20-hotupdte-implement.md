---
layout: post
title: lua closure热更新配置
published: true
---




的同时，使得修改后的lua代码立即生效，省得去重启客户端看效果，能够提高开发的效率。


我们县

###代码
代码在此：[lua_hotupdate](https://github.com/asqbtcupid/lua_hotupdate)
有4个文件，`luahotupdate.lua`和`hotupdatelist`是主要代码，`main.lua`和`test.lua`用来简单地测试说明这个东西怎么用。


把这4个文件放到同一目录下，`main.lua`是入口

	--main.lua
    local hotupdate = require "luahotupdate"
    local test = require "test"
    
    hotupdate.Init({"D:\\ldt\\workspace\\hotimplement\\src"}, "hotupdatelist")
    local run_times = 0
    local last_time = os.clock()
    
    while true do
      local now_time = os.clock()
      if now_time - last_time > 3 then
        last_time = now_time
        run_times = run_times + 1
        
        hotupdate.Update()
        test.func()
        
        if run_times >= 10 then
          break
        end
      end
    end

main会每三秒调用一次test.func，共计调用10次，在此循环期间，可以修改test.func的内容，并且会生效。
    
###接口说明
主要起作用的是luahotupdate，它有两个接口：

- Init(RootPath, UpdateListFile [, FailNotify])
- Update()

Init负责初始化，RootPath是你的lua文件目录，在本例里是D:\\ldt\\workspace\\hotimplement\\src，也就是放我这4个代码的地方。UpdateListFile是一个lua路径，要求这个lua文件返回一个table，这个table包含想要热更新的文件的文件名。FailNotify是热更新出错时的函数，需要该函数接受一个字符串参数，该字符串包含了出错的原因。

Update每运行一次就对hotupdatelist里面的文件进行热更新。


###注意事项
热更新只会更新函数的逻辑，而不更新“数据”，你可以改变里面函数的逻辑，比如test.lua

    local test = {}
    local times = 0
    
    local function upvalue_func()
      print("upvalue func")
    end
    
    function test.func()
      times = times + 1
      print("func", times)
      upvalue_func()
    end
    
    return test
    
upvalue_func和test.func都可以尽情修改，例如修改print别的字符串，你能看到循环里执行到新的代码。下图是个例子。

![例子动图]({{site.baseurl}}/images/hotupdate-example.gif)

它虽然目前也还有些问题，但已经运用都公司的项目里了。如果大家感兴趣，我会再发几篇文章说明它的原理。
