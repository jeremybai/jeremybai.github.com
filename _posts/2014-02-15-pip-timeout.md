---
layout: post
title: "pip下载超时的解决办法"
description: ""
categories: 
- python
tags: []
---
{% include JB/setup %}

　　使用pip安装包的时候经常会出现超时安装不了的情况，搜了下有两种方法可以解决：

## 1 指定源地址 ##
使用-i参数指定url，例如：  
  
    pip install flask -i http://pypi.v2ex.com/simple

## 2 修改配置文件 ##
~/.pip/文件夹下如果没有配置文件，就新建pip.conf配置文件：  
  
    [global]  
    timeout = 6000 
    index-url = http://pypi.v2ex.com/simple
    [install] use-mirrors = true  
    mirrors = http://pypi.v2ex.com/

当用v2ex的源也超时的时候再换使用默认源试试。