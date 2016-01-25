---
layout: post
title: lua closure热更新配置
published: true
---

本篇是我的lua热更新机制的配置教程，这里的“热更新”指的是调试开发代码的热更新，而不是线上项目代码的热更新，简单来讲就是，它能让你在开着客户端的同时，使得修改后的lua代码立即生效，省得去重启客户端看效果，能够提高开发或者找BUG的效率。


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
 
主要起作用的是luahotupdate，它有两个接口：
- Init(RootPath, UpdateListFile [, FailNotify])
- Update()

Init负责初始化，RootPath是你的lua文件目录，UpdateListFile是一个lua路径，要求这个lua文件长得跟hotupdatelist.lua一样返回一个table，这个table包含想要热更新的文件的文件名。FailNotify是热更新出错时的函数，需要该函数接受一个字符串参数，该字符串包含了出错的原因。

