---
layout: post
title: "python爬虫获取豆瓣电影——Beautiful Soup使用"
description: ""
categories: 
- python
- web
tags: [豆瓣,API]
---

　　在获取了网页的响应之后我们需要从整个页面中提取出我们需要的信息，可能是匹配某个div或者某个超链接等等，使用正则表达式是个不错的选择，但是写起来较为负责而且容易出错，这里我们使用Beautiful Soup作为提取页面信息的工具，Beautiful Soup 是一个可以从HTML或XML文件中提取数据的Python库，使用命令`pip install beautifulsoup4`进行安装，在PyPi中还有一个名字是BeautifulSoup的包，那是Beautiful Soup3的发布版本,因为很多项目还在使用BS3, 所以BeautifulSoup包依然有效。但是如果你在编写新项目,那么应该安装Beautiful Soup4。下面的用法摘自Beautiful Soup的[官方文档](http://beautifulsoup.readthedocs.org/zh_CN/latest/#id28)，有兴趣的可以自己看看，这里只列出一些基本用法。  
　　使用下面的一段HTML代码作为材料进行用法的介绍。这是《爱丽丝梦游仙境的》的一段内容:  

	html_doc = """
	<html><head><title>The Dormouse's story</title></head>
	<body>
	<p class="title"><b>The Dormouse's story</b></p>
	
	<p class="story">Once upon a time there were three little sisters; and their names were
	<a href="http://example.com/elsie" class="sister" id="link1">Elsie</a>,
	<a href="http://example.com/lacie" class="sister" id="link2">Lacie</a> and
	<a href="http://example.com/tillie" class="sister" id="link3">Tillie</a>;
	and they lived at the bottom of a well.</p>
	
	<p class="story">...</p>
	"""
　　首先使用BeautifulSoup解析这段代码,能够得到BeautifulSoup对象,并能按照标准的缩进格式的结构输出，其中第二个参数指定解析器，如果你没有安装lxml的话（`pip install lxml`），可以选择不写第二个参数:    

	from bs4 import BeautifulSoup
	soup = BeautifulSoup(html_doc, "lxml")
	
	print(soup.prettify())
	# <html>
	#  <head>
	#   <title>
	#    The Dormouse's story
	#   </title>
	#  </head>
	#  <body>
	#   <p class="title">
	#    <b>
	#     The Dormouse's story
	#    </b>
	#   </p>
	#   <p class="story">
	#    Once upon a time there were three little sisters; and their names were
	#    <a class="sister" href="http://example.com/elsie" id="link1">
	#     Elsie
	#    </a>
	#    ,
	#    <a class="sister" href="http://example.com/lacie" id="link2">
	#     Lacie
	#    </a>
	#    and
	#    <a class="sister" href="http://example.com/tillie" id="link2">
	#     Tillie
	#    </a>
	#    ; and they lived at the bottom of a well.
	#   </p>
	#   <p class="story">
	#    ...
	#   </p>
	#  </body>
	# </html>
　　下面列出几个简单的浏览结构化数据的方法:     
 
	soup.title
	# <title>The Dormouse's story</title>
	
	soup.title.name
	# u'title'
	
	soup.title.string
	# u'The Dormouse's story'
	
	soup.title.parent.name
	# u'head'
	
	soup.p
	# <p class="title"><b>The Dormouse's story</b></p>
	
	soup.p['class']
	# u'title'
	
	soup.a
	# <a class="sister" href="http://example.com/elsie" id="link1">Elsie</a>
	
	soup.find_all('a')
	# [<a class="sister" href="http://example.com/elsie" id="link1">Elsie</a>,
	#  <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>,
	#  <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>]
	
	soup.find(id="link3")
	# <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>
　　如果你希望从文档中找到所有`<a>`标签的链接，可以使用下面的方法，这种方法使用的较为广泛，在我们之后的项目中也会用到:    

	for link in soup.find_all('a'):
	    print(link.get('href'))
	    # http://example.com/elsie
	    # http://example.com/lacie
	    # http://example.com/tillie
　　或者从文档中获取所有文字内容:

	print(soup.get_text())
	# The Dormouse's story
	#
	# The Dormouse's story
	#
	# Once upon a time there were three little sisters; and their names were
	# Elsie,
	# Lacie and
	# Tillie;
	# and they lived at the bottom of a well.
	#
	# ...
　　上面列出了一些简单的Beautiful Soup用法，如果你想要快速上手，看完上面的例子你应该大概有个印象Beautiful Soup是如何使用的。如果你还需要更详细的使用方法，参考Beautiful Soup的[官方文档](http://beautifulsoup.readthedocs.org/zh_CN/latest/#id28)。   