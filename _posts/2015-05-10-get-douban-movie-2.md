---
layout: post
title: "python爬虫获取豆瓣电影——基本思路"
description: ""
categories: 
- python
- web
tags: [豆瓣,API]
published: false
---

##1 思路
　　我的目的是将尽可能多的电影信息从豆瓣电影中爬下来，首先我们需要做的就是起始url，结果浏览豆瓣电影相关的页面，我选择了以**分类**为入口，也就是`http://movie.douban.com/tag/`这个页面，以爱情这个分类为例，它的url是：  
　　`http://www.douban.com/tag/爱情/movie`  
　　每个页面上有15个电影，点击下一页之后，页面跳转到：   
　　`http://www.douban.com/tag/爱情/movie?start=15`  
　　再次点击下一页，页面跳转到：    
　　`http://www.douban.com/tag/爱情/movie?start=30`  
　　是不是觉得似乎发现了什么，没错，url的前面大部分信息是没有变化的，只是`start=`后面的数字发生了变化，每次增加了15，至于为什么是15，因为每个页面上有15个电影。所以我们只需要从`http://www.douban.com/tag/爱情/movie?start=0`开始，爬完这个页面之后就将`start=`后面的数字加上15便可以实现翻页的功能。   
　　翻页的功能解决之后接下来就是获取这个页面上列出的15个电影的相关信息了。以`http://www.douban.com/tag/%E7%88%B1%E6%83%85/movie?start=0`这个页面为例，我们通过chrome浏览器的开发者工具查看页面，如下图所示：  
[![1](http://7fv9jl.com1.z0.glb.clouddn.com/2015-05-10-get-douban-movie-2-1.jpg)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-05-10-get-douban-movie-2-1.jpg)     
　　从页面源代码中我们可以看出，页面上的15本书都在`<div class='mod movie-list'></div>`这个div中，每本书都对应一个`<dl>`，如下图所示：  
[![2](http://7fv9jl.com1.z0.glb.clouddn.com/2015-05-10-get-douban-movie-2-2.jpg)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-05-10-get-douban-movie-2-2.jpg)    
　　在`<dl>`中有用的信息主要有：电影的url、评分以及简要分类和演员信息，可是这些信息太笼统了，我想获得更加详细的信息，包括演员、国家、简介等信息。此时有两种方法可以获取：  
`——plan A：通过已有的url，跳转到电影介绍的相关页面，爬取相关的内容； ` 
`——plan B：通过调用豆瓣的API获取电影的相关信息。`   
　　这两种方法我都试了下，由于在电影介绍的相关页面的源代码中获取电影相关信息比较麻烦，所以采用直接调用豆瓣的API的方式获取信息，豆瓣API返回的信息为JSON格式，选取其中部分信息存储至数据库中就可以了。  


　　 