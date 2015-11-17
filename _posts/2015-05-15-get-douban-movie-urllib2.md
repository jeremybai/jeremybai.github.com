---
layout: post
title: "python爬虫获取豆瓣电影——urllib2使用"
description: ""
categories: 
- python
- web
tags: [豆瓣,API]
---

　　我们需要使用一些库来帮助我们更好的完成爬虫的功能，urllib2和BeautifulSoup是最主要的两个，这篇文章介绍下urllib2的使用。     
## urllib2
　　urllib2是一个可以帮助我们打开url、获取其中的内容的库。  
###1 简单使用
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
###2 设置超时处理
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
###3 伪装成浏览器
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
###4 异常处理
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
###5 使用代理
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
###6 使用cookie
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
###7 获取gzip压缩格式
　　网站的访问速度是由许多因素所共同决定的，包括程序的响应速度、网络带宽、服务器性能、与客户端之间的网络传输速度等等，除了这些因素之外，网页压缩也是一个重要的手段，重点是它完全不需要任何的成本，现在的网页普遍支持gzip压缩，这往往可以解决大量传输时间，以豆瓣的主页为例，我们通过程序查看压缩前和压缩之后响应数据的大小：  
{% highlight python %}  
import urllib
import urllib2

url = 'http://www.douban.com'
header = {'User-Agent': 'Mozilla/5.0 (Windows NT 6.2) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.12 Safari/535.11' }
req = urllib2.Request(url, headers=header)
null_proxy_handler = urllib2.ProxyHandler({})
opener = urllib2.build_opener(null_proxy_handler)
response = opener.open(req, timeout=3)
print '压缩前大小：' + response.info().get('Content-Length') + '字节'
req.add_header('Accept-encoding', 'gzip')
response = opener.open(req, timeout=10)
print '压缩后大小：' + response.info().get('Content-Length') + '字节'
{% endhighlight %} 
　　你也可以在浏览器中通过打开控制台查看页面的response header中Content-Length这一项直接查看response的大小，需要注意的是，通过程序获取的Content-Length可能和你在浏览器中看到的Content-Length大小略有偏差，可能是因为在浏览器中你是有登录信息的，但是在使用程序时没有使用登录信息所以导致response的大小略有不同程序。上面的程序运行结果为：

	压缩前大小：94700字节
	压缩后大小：17767字节  
　　可以看出，压缩之后response的大小为未压缩的1/5左右，这就意味着爬虫爬取的速度又有了提升的空间。然而urllib/urllib2库默认都不支持压缩，要返回压缩格式，必须在request的header里面面将'accept-encoding'设置为你支持的压缩类型，如gzip, deflate等等，然后读取response后检查header查看是否有'content-encoding'这一项来判断是否需要进行解码：  
{% highlight python %}
import urllib
import urllib2
from StringIO import StringIO
import gzip


url = 'http://www.douban.com'
header = {'User-Agent': 'Mozilla/5.0 (Windows NT 6.2) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.12 Safari/535.11' }
req = urllib2.Request(url, headers=header)
req.add_header('Accept-encoding', 'gzip')
null_proxy_handler = urllib2.ProxyHandler({})
opener = urllib2.build_opener(null_proxy_handler)
response = opener.open(req, timeout=10)
if response.info().get('Content-Encoding') == 'gzip':
    buf = StringIO(response.read())
    f = gzip.GzipFile(fileobj=buf)
    data = f.read()
else:
    data = response.read()
plain_text = str(data)
print plain_text  
{% endhighlight %} 
