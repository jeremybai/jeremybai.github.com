---
layout: post
title: "python爬虫获取豆瓣电影——准备工作"
description: ""
categories: 
- python
- web
tags: [豆瓣,API]
published: false
---

#1 什么是爬虫
　　爬虫的英文叫Spider，这个名字还是蛮恰当的，把你的程序比作一个虫子在互联网上爬，每爬一个页面都会存储下来。在抓取网页内容的过程当中，不断从当前页面上抽取新的url作为新的入口,直到满足一定的条件之后停止，这个过程类似于图的遍历。搜索引擎就是一个功能强大的爬虫，它爬取互联网上所有的信息。而对于具体功能的爬虫来说，只是需要从指定的页面中去获取希望获取的信息，比如说从豆瓣电影上获取所有电影的相关信息、从大众点评上获取某个地方的所有商铺的信息等等。在这几篇文章中我会介绍我在爬取豆瓣电影上所有电影相关信息并存储到本地的过程中使用的一些工具以及相关的思路。思路还在不断的改进中，也希望看到的朋友如果有什么更好的建议可以留言给我，毕竟这个脚本还是too naive了。  
　　这篇文章主要是一些预备知识的介绍，包括浏览页面的过程以及相关一些库的使用。  	  
#2 浏览器浏览页面发生了什么
　　[stackoverflow上有个稍微详细一点的回答](http://stackoverflow.com/questions/2092527/what-happens-when-you-type-in-a-url-in-browser)：  

	1 浏览器检查缓存；如果请求在缓存中，跳至9；  
	2 浏览器询问操作系统需要访问的服务器的ip；  
	3 操作系统通过DNS查找需要访问网站的ip并返回给浏览器；  
	4 浏览器与服务器之间建立TCP连接；  
	5 浏览器通过TCP连接向服务器提交HTTP请求；  
	6 浏览器接收到服务器发送过来的HTTP响应，TCP连接可能会被关闭或者被下次请求继续使用；    
	7 浏览器检查返回的响应是否被重定向（返回的状态码为3XX），授权请求（401）、错误（4XX和5XX）等等，这些响应的处理与正常响应的处理不一样（2XX）；  
	8 如果可以缓存的话，响应会被存储到缓存中；   
	9 浏览器将响应进行解码（比如说如果响应可能是使用gzip压缩的）；  
	10 浏览器决定如何处理响应（比如说这是一个HTML页面、图像或者音频等等）；  
	11 浏览器对响应进行渲染，或者对未识别类型的响应提高下载选项。  
　　简单的来说，就是（1）浏览器通过url找到需要访问的服务器的ip，向服务器提交一个请求；（2）服务器对这个请求进行响应，浏览器将响应进行解析渲染向用户进行展示。这个听起来蛮简单的过程（实际上巨复杂）也就是我们编程时的步骤。在下面一节中会介绍如何使用一些库来实现这个过程。    
#3 库的使用
　　我们需要使用一些库来帮助我们更好的完成爬虫的功能，urllib2和BeautifulSoup是最主要的两个。   
##3.1 urllib2
　　urllib2是一个可以帮助我们打开url、获取其中的内容的库。  
###3.1.1 简单使用
　　urllib2用法可以通过下面的代码进行演示： 
{% highlight python %}
import urllib2;

response = urllib2.urlopen("http://www.douban.com");
plain_text = response.read();
print plain_text;
{% endhighlight %}
　　运行终端程序，将会在输出窗口输出一大串代码，这段代码就是豆瓣首页的html代码，通过将需要访问的url作为参数传给`urlopen`函数，该函数通过解析url并向豆瓣的服务器提出请求，豆瓣服务器获得请求之后将响应的内容返回到本地，也就是response对象中，通过read函数将其中的内容读取出来。  
　　除了使用urlopen直接打开url之外，还有另外一种使用方法，通过将Request对象传给urlopen函数进行url的访问，代码如下：   
{% highlight python %}
import urllib2

req = urllib2.Request('http://www.douban.com')
response = urllib2.urlopen(req)
plain_text = response.read()
print plain_text
{% endhighlight %}
　　首先将url传入Request类的构造函数生成Request对象，将Request对象传入urlopen向url进行请求。这两种方法本质是相同的，因为在使用urlopen直接打开url时，urlopen内部调用opener.open函数，在opener.open中会判断传入的url参数是string类型还是Request类型，如果是string，则调用Request的构造函数将url转换为Request对象，opener.open的部分代码如下所示：   
{% highlight python %}
def open(self, fullurl, data=None, timeout=socket._GLOBAL_DEFAULT_TIMEOUT):
    # accept a URL or a Request object
    if isinstance(fullurl, basestring):
        req = Request(fullurl, data)
    else:
        req = fullurl
        if data is not None:
            req.add_data(data)
{% endhighlight %}
　　urlopen的参数还有许多，它的函数原型如下所示：  
{% highlight python %}
def urlopen(url, data=None, timeout=socket._GLOBAL_DEFAULT_TIMEOUT,
                cafile=None, capath=None, cadefault=False, context=None):
{% endhighlight %}
　　下面我们会对这些参数进行介绍。  
###3.1.2 设置超时处理
　　在urlopen的参数中有个`timeout`参数，这个参数决定了在请求服务器时等待的时间，如果设置为5，也就意味着如果等待了5s服务器还没有返回响应，那么就重新提交请求。默认timeout是设置为`socket._GLOBAL_DEFAULT_TIMEOUT`，这个值是socket全局的TIMEOUT时间。  
　　除了在这个函数中通过timeout参数值设置之外，还可以通过设置全局的timeout时间来设置：  
{% highlight python %}
import urllib2
import socket

# 5秒钟后超时重新连接
socket.setdefaulttimeout(5) 
# 或者
urllib2.socket.setdefaulttimeout(10)  
{% endhighlight %} 
###3.1.3 伪装成浏览器
　　urlopen向服务器提交访问请求时默认的http header为`Python-urllib/x.y`(x和y分别是大小版本号，比如说我现在使用的是Python 2.7，我的http header便是Python-urllib/2.7），有些服务器从header中分辨出来这个请求是由程序发起的而并非人为点击的，因此可能会拒绝响应，此时我们便需要对我们的请求进行伪装，通过将其伪装为浏览器的访问行为：   
{% highlight python %}
import urllib2

url = 'http://www.douban.com'
headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 6.2) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.12 Safari/535.11' }
req = urllib2.Request(url, headers)
response = urllib2.urlopen(req)
plain_text = response.read()
print plain_text
{% endhighlight %} 
　　其中user_agent代表了伪造的浏览器相关版本信息。  
###3.1.4 异常处理
　　在访问页面时经常会出现页面不存在或者其它一些错误，如果不对这些错误进行处理，就会使得程序终止运行。因此对于异常的捕捉和处理是比较重要的。最常见的错误主要有：URLError错误和HTTPError错误。  
　　URLError错误可能是由于你没有连接上因特网或者服你需要访问的服务器不存在或者不可达。  
　　HTTPError是URLError的子类，是服务器响应之后将应答传送到客户端时返回的出错的状态码。常见的状态码有200（请求成功）、404（页面没有找到）等等，其中100-299范围的状态码指示成功，3开头的状态码（3XX）对应重定向相关的错误，可以被urllib2处理，因此你只能看到400-599的错误状态码，更多的状态码可以查看[HTTP状态码的wiki](https://zh.wikipedia.org/zh-sg/HTTP%E7%8A%B6%E6%80%81%E7%A0%81)。  
　　使用try...catch语句对错误进行捕捉并打印出相关的信息：  
{% highlight python %}
import urllib2

url = 'http://www.douban.com/info'
headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 6.2) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.12 Safari/535.11' }
req = urllib2.Request(url, headers=headers)
try:
    response = urllib2.urlopen(req, timeout=3)
    plain_text = response.read()
    print plain_text
except urllib2.HTTPError as e:
    print e.code, e.reason
except urllib2.URLError as e:
    print e.reason
{% endhighlight %} 
　　reason是父类URLError的属性，代表错误的原因。因为这个页面时不存在的，因此会产生404 HTTPError。     
　　需要注意的是在捕捉URLError错误时不要试图获取它的状态码，因为URLError是没有状态码的。  
　　还有个需要注意的地方就是因为HTTPError是URLError的子类，因此在进行一场捕捉的时候将子类的异常捕捉（也就是HTTPError）写在父类（HTTPError）的前面，这样当子类捕捉不到错误时再由父类进行捕捉，不然子类的错误也会被父类捕捉。  
###3.1.5 使用代理
　　当爬虫爬取的速度太快时可能会导致服务器将你的ip封掉，也就意味着除非你换了ip，不然无法从服务器再获得数据。显然如果你需要真的改变你的ip是比较麻烦的，你可能需要将家里的光猫重新启动，使它重新被分配一个地址这样你的地址才可能发生变化。幸运的是urllib2提供了代理的支持使得你只需要提供代理的ip和端口便可以通过代理ip进行访问而不需要去改变你本机的ip：  
{% highlight python %}
url = 'http://www.douban.com'
headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 6.2) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.12 Safari/535.11' }
req = urllib2.Request(url, headers=headers)
try:
    proxy_handler = urllib2.ProxyHandler({'http': '180.166.112.47:8888'})
    opener = urllib2.build_opener(proxy_handler)
    plain_text = opener.open(req, timeout=3).read()
    print plain_text
except urllib2.HTTPError as e:
    print e.code, e.reason
except urllib2.URLError as e:
    print e.reason 
{% endhighlight %}  
###3.1.6 使用cookie
　　如果你想要通过urllib2获取cookie，那么你就需要cookielib库的帮助，下面的代码是获得了豆瓣的cookie并且打印出cookie的每一项的名称和值：   
{% highlight python %}  
import urllib2
import cookielib

cookie = cookielib.CookieJar()
opener = urllib2.build_opener(urllib2.HTTPCookieProcessor(cookie))
response = opener.open('http://www.douban.com/')
for item in cookie:
    print item.name, item.value
{% endhighlight %} 
##3.2 Beautiful Soup
　　在获取了网页的响应之后我们需要从整个页面中提取出我们需要的信息，可能是匹配某个div或者某个超链接等等，使用正则表达式是个不错的选择，但是写起来较为负责而且容易出错，这里我们使用Beautiful Soup作为提取页面信息的工具，Beautiful Soup 是一个可以从HTML或XML文件中提取数据的Python库，使用命令`pip install beautifulsoup4`进行安装。  
###3.2.1 
　　
###3.2.2
　　
###3.2.3
　　

##3.3 多线程
　　由于GIL的存在导致在任何时刻只有一个线程在执行，但是这并不意味着多线程编程在Python中一无是处，当爬虫的多个线程的其中一个在爬取网站时阻塞在等待服务器的响应时，其他线程的任务便可以利用阻塞任务等待的时间进行其他操作以提高效率。操作数据库时也存在同样的问题，有可能你需要等待数据库返回的结果，在等待期间你可以利用这段时间做其它的一些操作。    


　　 