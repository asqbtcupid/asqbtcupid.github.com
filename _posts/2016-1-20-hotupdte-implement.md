---
layout: post
title: lua closure热更新配置
published: true
---


本篇教你使用我的“热更新”，这里的热更新是指，lua虚拟机运行时，你去修改代码，新代码会替代老代码生效，这有两方面的好处。对线下开发项目来说，省去重启客户端看效果，能够提高开发的效率。对于线上项目来说，可以不停服更新。我们来看个例子吧：

![例子动图]({{site.baseurl}}/images/hotupdate-example.gif)

通过上面的例子，你也许观察到了函数逻辑被热更新了，而数据没有被热更新，就是test.count，局部变量count，全局变量d_count还维持原来的值，这是涉及一个非常重要的原则：
##热更新不破坏代码的逻辑
这是什么意思呢？比如说上面的例子，count是用来统计test.func()运行了多少次，我们去热更新test.func()也许是为了修它的某些bug，然而这个count值应该保持正确，同理还有test.count和d_count。当然我们也可以在热更新时修改它们的值，就如例子最后把count置零。

可能你还注意到了另一个细节：被更新的test文件，并没有为支持热更新额外写了多余的代码。这是另一个重要的原则：
##热更新不被觉察
我们实现一个热更新机制可以有很多办法，曾经有的朋友跟我讨论，说他们的模块为了支持数据不变的特性，需要在模块里额外写一些代码来记录旧值，热更新之后再把旧值copy过来，或者用一些特殊的语法来支撑。这些方式弊端太多？想象一下，你带着你的热更新机制来到了一个新的项目里，这个项目已经写了成百上千的模块，这时你难道要一一修改这些模块来支持热更新吗？再想象一下，同事为了支持你的热更新系统，必须遵循一些代码规则，这无形中加重了团队的负担。

“不被察觉”指的是，我们写我们的逻辑代码，该怎么写就怎么写，想怎么写就怎么写，就像我们不知道有热更新这回事。但是不管代码写成什么样，都能被热更新。我的热更新机制可以随意用到什么项目上，对于％99.9的模块来说，都能够实现热更新，不需要你修改这修改那来迎合热更新。




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
