---
layout: post
title: "如何解决SecureCRT自动断开的问题"
description: ""
categories: 
- web
- linux
tags: [SecureCRT]
---

　　使用SecureCRT时经常会出现停止操作一段时间中之后就需要再次连接的问题，操作几次之后觉得有些麻烦，便看看是否有解决的办法：  
　　CRT配置方法：
会话选项–> 终端–> 反空闲–> 发送字符串可以设置，比如发送\n 、null或其他信息过去，后面可以设置每隔多少秒发送，比如可以3000秒一次，这样可以保证不会掉线。