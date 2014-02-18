---
layout: post
title: "自动登录网关的python脚本"
description: ""
categories: 
- python
- web
tags: [苏州大学,网关]
---
{% include JB/setup %}
### 摘要 ：
　　*这篇文章主要介绍如何使用python编写登陆苏州大学网关的脚本。该脚本每隔一秒检测网络是否连接，如果没有联网就会自动登陆网关，若网络没有断开则继续判断*
 
# 1 准备工作 
## 1.1 工作原理 
　　首先我们先分析一下我们登陆网关的流程是怎么样的。当我们打开wg.suda.edu.cn这个网站时，选择了登陆时间和免费网址等几个选项，再点击“登陆网关”的时候，就会产生post请求，那么是什么post请求呢？这就要说到超文本传输协议（HTTP）了，HTTP的设计目的是保证客户机与服务器之间的通信，它的工作方式是客户机与服务器之间的请求-应答协议。web浏览器可能是客户端，而计算机上的网络应用程序也可能作为服务器端。在客户机和服务器之间进行请求-响应时，GET和POST方法是两种最常被用到的http请求方法。POST方法用于向指定的资源提交要被处理的数据，GET方法从指定的资源请求数据。在我们点击“登陆网关”的时候，产生的post请求包含的信息的大概含义就是我们的浏览器（也就是客户端）对网络中心的服务器说：我需要2个小时的上网时间。服务器就会根据这个请求执行相应的操作。  
## 1.2 伪造数据 
　　我们要做的就是伪造浏览器提交的post数据，那么如何查看浏览器提交的数据呢？以chrome为例，打开wg.suda.edu.cn，按F12，刷新页面，点击到Network标签下我们就可以看到产生的HTTP 方法了
![](http://d.pcs.baidu.com/thumbnail/af8dc6d1b9b7c41b377a21d229ff5654?fid=859179042-250528-99345073&time=1392713681&rt=pr&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-fsZoY2mVluuPNjDnHa7SkqGnLrw%3D&expires=8h&prisign=RK9dhfZlTqV5TuwkO5ihMQzlM241kT2YfffnCZFTaEPwOxHv/XxtwRXLxDSXMBba1Ms9seOiqT9/QffwI8K2Baw0mmLABRQNl51b/oS8+InqoadADmwcyvUvQUkAvl8j879BObtgHePZ09BZiFo3CF7dwGcK/xIzausW8RTta6nycgI5A4W/Ju5XK/lbx40b7xvlT+8yG2JtdgCRiVnR1ML4mUrvg32eSuq8iSxudjQ=&r=250287517&size=c850_u580&quality=100) 
　　输入你的账号和密码，点击登陆网关会发现有个post请求，这个就是我们要找的东西了！
![](http://d.pcs.baidu.com/thumbnail/69c21168b55a4fb4d35ea56d9102b38e?fid=859179042-250528-3942476861&time=1392713681&rt=pr&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-KhRgRucLGYjpAw4o%2BkbYr%2BfQ7sg%3D&expires=8h&prisign=RK9dhfZlTqV5TuwkO5ihMQzlM241kT2YfffnCZFTaEPwOxHv/XxtwRXLxDSXMBba1Ms9seOiqT9/QffwI8K2Baw0mmLABRQNl51b/oS8+InqoadADmwcyvUvQUkAvl8j879BObtgHePZ09BZiFo3CF7dwGcK/xIzausW8RTta6nycgI5A4W/Ju5XK/lbx40b7xvlT+8yG2JtdgCRiVnR1ML4mUrvg32eSuq8iSxudjQ=&r=744732687&size=c850_u580&quality=100) 
　　点击这个post数据，找到Request Header和Form Data，这两项是我们着重关注的。
![](http://d.pcs.baidu.com/thumbnail/b012772f034f1ec9c309613c7754b91a?fid=859179042-250528-346253091&time=1392713681&rt=pr&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-gVZtToUfGXT3SguRcyzQfeVGFlY%3D&expires=8h&prisign=RK9dhfZlTqV5TuwkO5ihMQzlM241kT2YfffnCZFTaEPwOxHv/XxtwRXLxDSXMBba1Ms9seOiqT9/QffwI8K2Baw0mmLABRQNl51b/oS8+InqoadADmwcyvUvQUkAvl8j879BObtgHePZ09BZiFo3CF7dwGcK/xIzausW8RTta6nycgI5A4W/Ju5XK/lbx40b7xvlT+8yG2JtdgCRiVnR1ML4mUrvg32eSuq8iSxudjQ=&r=142407762&size=c850_u580&quality=100) 
　　我们已经看到了看到了我们提交的post数据了，现在我们就开始构造我们的post数据，我们需要将请求的数据构造成一个字典（为什么构造成字典接下来会有介绍）。

    data = {
    		'TextBox1' : self.user,  #用户名
    		'TextBox2' : self.pwd,   #密码
    		'nw' : 'RadioButton1',   #免费地址
    		'tm' : 'RadioButton6',	 #登陆2个小时
    		'Button1' : u'登录网关'.encode('gb2312'),
    		'__VIEWSTATE' :'/wEPDwUKMTM2NjA4NzMwMw8WAh4IcGFzc3dvcmQFCjUwNjUxMjV6eGMWAgIBD2QWDAIBDxYCHgRUZXh0ZWQCAw8PFgIfAWVkZAIFDxYCHwFlZAIHDxYCHgdWaXNpYmxlZ2QCCQ8WAh8CaGQCCw8WAh8BBTDlvZPliY3lnKjnur/kurrmlbA6PGIgc3R5bGU9J2NvbG9yOnJlZCc+MjA5OTwvYj5kGAEFHl9fQ29udHJvbHNSZXF1aXJlUG9zdEJhY2tLZXlfXxYIBQxSYWRpb0J1dHRvbjEFDFJhZGlvQnV0dG9uMgUMUmFkaW9CdXR0b24zBQxSYWRpb0J1dHRvbjQFDFJhZGlvQnV0dG9uNQUMUmFkaW9CdXR0b242BQxSYWRpb0J1dHRvbjcFDFJhZGlvQnV0dG9uOA==',
    		'__EVENTVALIDATION':'/wEWDQLF1pCSCQLs0bLrBgLs0fbZDALazsi9CQLazrwZAtTO6PQIAtTO/K8HAtTOgIoOAtTOlOUGAtTOuMANAtTOzLwEAoznisYGAoXZ9dsD'
    		}
　　这里插一句，在一开始调试的时候始终不能登录，原因就是在于构造的时候只写了前面5项，没有把`VIEWSTATE`和`EVENTVALIDATION`加上，后来实在不行，便抱着试试的心态把post的数据全部加上，发现就可以登录了=。=！。搜了下这个东西是干嘛的：VIEWSTATE是asp.net的服务器控件自动产生的，大家都知道，在asp时代，一个html控件的值,比如input 控件值,当我们把表单提交到服务器后, 页面再刷新回来的时候, input里面的数据已经被清空. 这是因为web的无状态性导致的, 服务端每次把html输出到客户端后就不再于客户端有联系.之后的asp.net巧妙的改变了这一点. 当我们在写一个asp.net表单时, 一旦标明了 `form runat=server` ,那么,asp.net就会自动在输出时给页面添加一个隐藏域  
`<input type="hidden" name="__VIEWSTATE" value="">`.  
　　有了这个隐藏域,页面里其他所有的控件的状态,包括页面本身的一些状态都会保存到这个控件值里面. 每次页面提交时一起提交到后台,asp.net对其中的值进行解码,然后输出时再根据这个值来恢复各个控件的状态.这篇文章介绍的稍微详细一点，感兴趣的点[这里](http://www.cnblogs.com/yzxchoice/archive/2006/09/08/498499.html)。
　　上面我们说有两项数据需要我们着重关注，除了Form Data（表单提交的数据）还有就是Request Header了，我们还需要伪造Request Header来将我们的脚本产生的请求伪装成是浏览器产生的。

    user_agent = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/29.0.1547.66 Safari/537.36'
    headers = {
    		  'Host'   : self.host,
    		  'Origin' : self.origin,  
    		  'User-Agent' : user_agent,
    		  }
　　对有些 header 要特别留意，服务器会针对这些 header 做检查.比如说User-Agent，有些服务器或 Proxy 会通过该值来判断是否是浏览器发出的请求。或者Content-Type（我这里没有使用），在使用REST接口时，服务器会检查该值，用来确定 HTTP Body 中的内容该怎样解析，如果服务器需要的数据是JSON类型，你却设置为了XML类型的数据传给服务器，服务器是肯定不会接受的。常见的取值有： 
 
    　　application/xml ： 在 XML RPC，如 RESTful/SOAP 调用时使用  
    　　application/json ： 在 JSON RPC 调用时使用  
    　　application/x-www-form-urlencoded ： 浏览器提交 Web 表单时使用  

　　在使用服务器提供的 RESTful 或 SOAP 服务时， Content-Type 设置错误会导致服务器拒绝服务。
# 2 发送请求 
　　这样，准备活动已经做完，接下来就要开始向服务器发送请求了。这里我们会用到[urllib2](http://docs.python.org/2/library/urllib2.html)这个模块，urlib2是使用各种协议完成打开url的一个扩展包。最简单的使用方式是调用urlopen方法，可以接受一个字符串型的url地址或者一个Request对象。将打开这个url并返回结果为一个像文件对象一样的对象.使用方法如下：

    import urllib2
    response = urllib2.urlopen('http://wg.suda.edu.cn/indexn.aspx')
    the_page = response.read()
    print the_page
　　我们说了，参数可以为url或者Request对象，所以也可以这样用：

    import urllib2
	request = urllib2.Request('http://wg.suda.edu.cn/indexn.aspx')
	response = urllib2.urlopen(request)
	the_page = response.read()
    print the_page
　　这两者的效果是一样的，在这里我们使用的是第二种方法，因为我们需要将构造的post数据和headers在请求url的同时发出去，于是我们先调用Request函数将url，postdata和headers构造为Request对象再使用urlopen函数打开这个Request对象。

    postdata = urllib.urlencode(data)
    request = urllib2.Request(self.url, postdata, headers)
    response = urllib2.urlopen(request)
　　urlencode函数将字典或者包含两个元素的元组列表转换成url参数。这里就是为什么我们要将之前的post的数据构造成字典了，因为这个函数的参数必须要是字典。
　　这个函数执行完毕之后，我们会发现原本断了的网关又重新连上了！！！  
　　这里再啰嗦一句，有同学可能会问这里的url为什么是`http://wg.suda.edu.cn/indexn.aspx`而不是`http://wg.suda.edu.cn`，原因就是`http://wg.suda.edu.cn`这个url被重定向到`http://wg.suda.edu.cn/indexn.aspx`这个url了，怎么可以知道url有没有被重定向呢？可以使用geturl()函数，检查一下 Response的UR和Request的URL是否一致就可以了。

    import urllib2
    response = urllib2.urlopen('http://www.google.cn')
    redirected = response.geturl() == 'http://www.google.cn'

# 3 其他功能 
　　基本功能已经实现了，接下里还需要加了其他工作：
## 3.1判断网络是否连通 
　　这里我通过ping百度的ip来测试是否断网（国内访问百度比较快，若有其他需求只需要改成你需要访问的ip即可）。设置超时时间为3秒，若超过三秒就会返回socket.timeout异常，捕捉到之后返回False。

	def internet_on():
	    try:
	        response=urllib2.urlopen('http://115.239.210.26',timeout=3) 
	        return True
	    except urllib2.URLError as e: 
	        print u"urllib2.URLError错误" 
	    except socket.error as e: 
			type, value, traceback = sys.exc_info()[:3] 
			if type == socket.timeout: 
				print u"socket.timeout错误" 
			else: 
				print u"其他socket错误"
	    return False
## 3.2 读取配置信息 
　　写完上面的功能，想把它发给同学用用看，刚准备发的时候，又想到一个问题，难道每个运行这个程序的人都要打开这个脚本来修改其中的用户名和密码吗？万一有些人不明白怎么修改怎么办？于是便想着能不能写一个简单的配置文件，用户只要修改这个配置文件将自己的用户名和密码写到配置文件中，程序只需要读取配置文件来获得用户名和密码就可以了。这就需要另外一个模块的帮助了：[ConfigParser](http://docs.python.org/2/library/configparser.html)。  
　　ConfigParser模块解析的配置文件的格式比较像ini的配置文件格式。  
　　ini 文件是文本文件，ini文件的数据格式一般为：

    [Section1 Name] 
    KeyName1=value1 
    KeyName2=value2 
    ...
    
    [Section2 Name] 
    KeyName1=value1 
    KeyName2=value2

　　文件中由多个section构成，每个section下又有多个配置项，比如：

    #配置文件，请将UserID和PassWord设置成你的网关的账号和密码
    [Info]
    UserID = 2012XXXXXXX
    PassWord = XXXX

　　这个ini文件就是我们使用的配置文件，其中只有一个session，名字叫做Info，里面有两组值，我们可以把它们看做字典，第一组键值对的键名叫UserID，存放学号，它的值为2012XXXXXXX，第二组键值对的键名叫PassWord存放密码，它的值为XXXX，（大家用自己的学号和密码代替就可以了）。接下里就要读取配置文件的信息了。

	try:
		# 创建SafeConfigParser对象
		config = ConfigParser.SafeConfigParser()
		# 获得当前路径接着读取配置文件
		config.read(os.path.dirname(os.path.abspath(__file__)) + '/UserInfo.ini')
		# 获取配置文件中的字段
		user = config.get('Info','UserID')
		pwd = config.get('Info','PassWord')	
	except ConfigParser.NoSectionError as e:
		print u'Error：用户信息未配置，请将您的学号和密码填入UserInfo.ini文件'
		while True:
			pass
　　我们除了读取配置文件的信息之外还可以像配置文件里面写信息，有兴趣的点击[这里](http://docs.python.org/2/library/configparser.html)。 
# 4 完成  
　　这样一个简单脚本就完成了,运行的结果如下：  
![]({{site.img_url}}/2014-01-20/4.JPG)    
　　当然还有很多容错方面的问题没有考虑到，如用户名密码错误怎么办，还有可以使用cookie来保存读取信息等等。接下里有时间慢慢改进吧。  
# 5 下载
　　完整代码点击[这里](https://github.com/jeremybai/AutoLoginGateWay)，如果电脑上没有装python的同学，点击这里  
　　[(32位系统版本)](http://pan.baidu.com/s/1i33bXYX)  
　　和  
　　[(64位系统版本)](http://pan.baidu.com/s/1mgG3f1u)  
　　有打包成exe的程序，只要解压参考README运行即可。
### 注意： ###
　　**编辑UserInfo.ini时不要使用微软的记事本，因为它在保存的时候会加上BOM头，导致读取文件出错，最好使用notepad++编辑完选择格式为UTF-8无BOM格式编码然后保存。否则程序无法运行。**
## 补 ##
### 2014.2.14 ###
　　现在不需要修改配置文件，只需要打开exe文件输入用户名和密码即可。