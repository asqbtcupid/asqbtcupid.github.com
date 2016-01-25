---
layout: post
title: lua closure热更新配置
published: true
---

本篇是我的lua热更新机制的配置教程，这里的“热更新”指的是调试开发代码的热更新，而不是线上项目代码的热更新，简单来讲就是，它能让你在开着客户端的同时，使得修改后的lua代码立即生效，省得去重启客户端看效果，能够提高开发或者找BUG的效率。

代码在此：[lua_hotupdate](https://github.com/asqbtcupid/lua_hotupdate)
有4个文件，主要代码在`luahotupdate.lua`和`hotupdatelist`,`main.lua`和`test.lua`用来简单地测试说明这个东西怎么用。
