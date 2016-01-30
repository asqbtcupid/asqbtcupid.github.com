---
layout: post
title: Lua热更新
published: true
---




本篇教你使用我的“热更新”，这里的热更新是指，lua虚拟机运行时，你去修改代码，新代码会替代老代码生效，这有两方面的好处。对线下开发项目来说，省去重启客户端看效果，能够提高开发的效率。对于线上项目来说，可以不停服更新。我们来看个例子吧：

（动图加载可能有点慢，请耐心一点儿～）

![例子动图]({{site.baseurl}}/images/hotupdate-example.gif)

通过上面的例子，你也许观察到了函数逻辑被热更新了，而数据没有被热更新，就是test.count，局部变量count，全局变量d_count还维持原来的值，这是涉及一个非常重要的原则：

##热更新不破坏逻辑
我觉得这是热更新机制的试金石，这是什么意思呢？最简单的测试方法就是，你修改了代码，但是其实逻辑没有变化，比如你在哪里加了一行无效的语句，local i ＝ 0，然后这个i从未被使用，这是一句废代码，启动热更新。更新成功之后你的系统的逻辑不应该受到任何一丁点儿的影响，因为你压根儿没改什么逻辑。

对于上面的例子来说，count是用来统计test.func()运行了多少次，我们去热更新test.func()也许是为了修它的某些bug，为了不破坏逻辑，需要这个count值在热更新前后保持一致，因为也许有别的逻辑依赖于这个值。同理还有test.count和d_count。当然我们也可以在热更新时主动修改它们的值，就如例子最后把count置零。

可能你还注意到了另一个细节：被更新的test文件，并没有为支持热更新额外写了多余的代码。这是另一个重要的原则：
##热更新不被觉察
我们实现一个热更新机制可以有很多办法，曾经有的朋友跟我讨论，说他们的模块为了支持数据不变的特性，需要在模块里额外写一些代码来记录旧值，热更新之后再把旧值copy过来，或者用一些特殊的语法来支撑。这些方式有很多弊端。想象一下，你带着你的热更新机制来到了一个新的项目里，这个项目已经写了成百上千的模块，这时你难道要一一修改这些模块来支持热更新吗？再想象一下，同事为了支持你的热更新系统，必须遵循一些代码规则，这无形中加重了他们的负担，和出错的概率。

“不被察觉”指的是，我们写我们的逻辑代码，该怎么写就怎么写，想怎么写就怎么写，就像我们不知道有热更新这回事。但是不管代码写成什么样，都能被热更新，至少绝大部分都可以。我的热更新机制对于％99的带来来说，都能够应用，至少我所在的项目里那2000多个lua文件都ok，不需要修改这修改那来迎合热更新。


###代码
代码在此：[lua_hotupdate](https://github.com/asqbtcupid/lua_hotupdate)
有4个文件，`luahotupdate.lua`和`hotupdatelist.lua`是主要代码，`main.lua`和`test.lua`用来简单地测试说明这个东西怎么用。

简单的做法是把这4个文件放到同一目录下，`main.lua`是入口

    local HU = require "luahotupdate"
    HU.Init("hotupdatelist", {"D:\\ldt\\workspace\\hotimplement\\src"}) --please 	 replace the second parameter with you src path
    
    function sleep(t)
      local now_time = os.clock()
      while true do
        if os.clock() - now_time > t then
          HU.Update() 
          return 
        end
      end
    end
   
   	local test = require "test"
    print("start runing")
    while true do
      test.func()
      sleep(3)
    end

main每三秒调用一次test.func()，可以修改test.func的代码，并且会生效。
    
###接口说明
主要起作用的是luahotupdate.lua，它有两个接口：

- Init(UpdateListFile, RootPath, [, FailNotify, ENV])
- Update()

Init负责初始化，各个参数说明：

1. UpdateListFile
是你的hotupdalist.lua的require路径，hotupdalist.lua只是返回一个table，这个table里记录着需要被热更新的文件名，在本例就是“test”，hotupdalist.lua也是可以在运行时修改，修改后不需要重启就生效。

2. RootPath
是一个table，这个table的每个项都是一个目录地址，该目录下包括子目录下的lua文件都可以被热更新，只需要把文件名填进hotupdatelist里。

3. FailNotify
需要传入一个函数，该函数可以接收一个字符串作为输入，当热更新出错时会把出错信息告知这个函数，你问我什么时候会出错呢？有几种情况，比如hotupdalist.lua里包含了不存在的文件名，又如修改后的文件有了语法错误导致无法编译。可以为nil。

4. ENV
需要传入环境表，lua5.1默认不传就是“_G”。

Update()是执行一次热更新，它会找到hotupdatelist里面的文件，热更新它们的函数。当然并不是里面定义的所有函数都会被热更新，由下面的细则说明。

###热更新细则
