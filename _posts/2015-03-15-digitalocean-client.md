---
layout: post
title: "DigitalOcean使用小记--使用shadowsocks进行科学上网"
description: ""
categories: 
- linux
- web
tags: [DigitalOcean,VPS]
---
{% include JB/setup %}
　　对比了Linode和DigitalOcean（简称DO），选择DO了，两者的套餐如图所示。  
[![1](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-1.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-1.png)   
　　选择了DO最便宜的套餐，试用了下，网上许多人推荐使用旧金山的机房，ping下来速度在400ms左右，觉得使用secureCRT登陆上去操作延迟较高，又试了下大多数人不推荐的新加坡的机房，平均174ms左右，速度还不错。下面介绍下DO的使用流程。顺便给个[邀请链接](https://www.digitalocean.com/?refcode=12e726830dc4)，感兴趣的可以从这个链接进去试试看。
#1 注册账号
　　首先，你需要[注册](https://www.digitalocean.com/?refcode=12e726830dc4)一个DO的账号，注册过程就不需要多说了，邮箱啥的填写好就可以了。注册完之后还没能开始使用，你需要绑定你的信用卡或者向你的账户中充值5刀才可以创建你的VPS，可以使用Paypal绑定银联的卡进行支付，支付完成之后便可以开始创建VPS了。  
#2 创建Droplet
##2.1 基本步骤
　　点击Create Droplet按钮，输入Droplet的名字（Hostname），选择Image，有Ubuntu、FreeBSD、CentOS等发行版本可以选择，我用的是CentOS7 64位，感觉作为RedHat一系，CentOS的稳定性可能会较高一些，Size选择最低的一档试试看，好用可以再进行扩容，机房选择的是Singapore机房，你可以选择其中一个创建完之后测下速度，不行的话再删掉重新创建一个Droplet，自己挑一个你访问速度较快的服务器。Available Settings根据自己的需求选择，学生的话IPV6可以选上。  
　　下面是Add SSH Keys，这个步骤是可选的，这里详细介绍下SSH密钥对（SSH Key），SSH密钥对可以让你方便的通过SSH登录到服务器，而无需输入密码，你不需要发送你的密码到网络中，SSH密钥对被认为是更加安全的方式。**SSH密钥对总是成双成对出现的，一把公钥，一把私钥。公钥可以自由的放在您所需要连接的SSH服务器上，而私钥必须稳妥的保管好。**所谓"公钥登录"，原理很简单，就是你将自己的公钥储存在远程主机上。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。这样子即可保证了整个登录过程的安全，也不会受到中间人攻击。
##2.2 生成密钥对
　　首次查看你的机器上有没有密钥对。查看你的/home目录下是否有隐藏的.ssh文件夹，其中是否有以pub为后缀的文件，有的话直接打开复制其中的内容即可，没有的话使用ssh-keygen命令生成密钥对。  
　　**PS: Windows用户查看`C:\Users\<你的用户名>\.ssh`文件夹下查看是否有以pub为后缀的文件，如果没有，安装[git for Windows](http://msysgit.github.io/)，安装完使用git bash输入下面相同的命令生成密钥对。**
  
    ssh-keygen -t rsa -C "你的邮箱地址"
　　完成之后就会生成密钥对（id_rsa和id_rsa.pub），复制公钥id_rsa.pub其中的内容填至表单中即可。最后点击创建等待几分钟即可完成VPS的创建，创建完成之后过几分钟你的邮箱会收到邮件，告诉你你的VPS的ip以及密码等。  
#3 安装shadowsocks
　　shadowsocks是一个可穿透防火墙的快速代理，我们可以在VPS上部署shadowsocks使得我们的设备通过VPS实现科学上网。
##3.1 安装
　　根据你的VPS使用的镜像选择不同的安装方法。  
　　**Debian / Ubuntu:**  

    apt-get install python-pip
    pip install shadowsocks
　　**CentOS:**  

    yum install python-setuptools && easy_install pip
    pip install shadowsocks
　　如果你的VPS上安装的是windows，稍微麻烦一siusiu，可以参考[这篇wiki](https://github.com/shadowsocks/shadowsocks/wiki/Install-Shadowsocks-Server-on-Windows)。
##3.2 启动
　　安装完成之后便可以启动shadowsocks。-p表示端口号，-k表示密码，-m表示加密方式。

    ssserver -p 8999 -k 123456 -m rc4-md5
　　如果要后台运行：

    sudo ssserver -p 8999 -k 123456 -m rc4-md5 -d start
　　如果要停止：

    sudo ssserver -d stop
　　如果要检查日志：

    sudo less /var/log/shadowsocks.log
　　用 -h 查看所有参数，如果你觉得输入命令比较麻烦的话，也可以使用配置文件进行配置。使用nano命令新建一个json文件。

    nano /etc/shadowsocks.json
　　输入相应的内容，**特别注意修改字段为你自己的设置，不要把下面的配置文件不修改直接拷贝**。

	{
    	"server":"128.199.166.172",
    	"server_port":8999,
    	"local_address": "127.0.0.1",
    	"local_port":1080,
    	"password":"<你的密码>",
    	"timeout":300,
    	"method":"aes-256-cfb",
    	"fast_open": false
	}
　　紧接着以该配置文件为参数运行命令:

    ssserver -c /etc/shadowsocks.json
　　后台运行的命令如下所示:

    ssserver -c /etc/shadowsocks.json -d start
    ssserver -c /etc/shadowsocks.json -d stop
#3.3 使用shadowsocks客户端
　　shadowsocks的客户端支持大多数的主流平台，一般需要配置一下服务器的ip地址和之前设置好的连接密码即可，参考下面的链接：    
- [Windows](https://github.com/shadowsocks/shadowsocks-csharp) / [OS X](https://github.com/shadowsocks/shadowsocks-iOS/wiki/Shadowsocks-for-OSX-Help)  
- [Android](https://github.com/shadowsocks/shadowsocks-android) / [iOS](https://github.com/shadowsocks/shadowsocks-iOS/wiki/Help)  
- [OpenWRT](https://github.com/shadowsocks/openwrt-shadowsocks)  
　　以Windows平台为例，从[SourceForge](http://sourceforge.net/projects/shadowsocksgui/files/dist/)上下载你的平台对应的版本号，Windows上根据你机器安装的.net版本选择对应的安装包，我的机器上因为安装了.net framework 4.0，因此下载了Shadowsocks-win-dotnet4.0-2.3.1.zip，解压之后运行Shadowsocks.exe，运行界面如下所示。  
[![2](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-2.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-2.png)  
　　填写好IP、端口号以及密码，这些都要与上述配置文件中一致，代理端口默认。保存之后，右击任务栏图标选择好设置的服务器并启动系统代理，注意一定要像下图中一样选中，即前面出现打勾选项。至此我们已经完成了客户端的设置，接下来进行浏览器的设置，设置完之后便可以科学上网了。  
[![3](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-3.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-3.png)  
　　OS X下的使用比较简单，直接下载对应的dmg文件进行安装，安装完填写好IP及端口等设置，同样选择好设置的服务器并选择打开shadowsocks，同样的注意点，一定要选中，即前面出现打勾选项。此时应该就可以科学上网了。
#3.4 浏览器设置
　　浏览器以chrome为例，如果用过GoAgent的话就会发现设置比较类似，下载并安装Proxy SwitchySharp插件，打开Proxy SwitchySharp的设置，新建一个情景模式，命名为 Shadowsocks，设置好端口，**注意socks模式选择socks5**，如图所示。    
[![4](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-4.jpg-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-4.jpg)  
　　接着点击切换规则，如下图所示，启用切换规则，然后在 URL 模式里输入\*google\*通配Google的所有域名，模式匹配选择通配符，情景模式为Shadowsocks，然后把在线规则列表打钩，输入：

    https://autoproxy-gfwlist.googlecode.com/svn/trunk/gfwlist.txt
　　勾选AutoProxy兼容列表，点立即更新并保存。  
[![5](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-5.jpg-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-5.jpg)  
　　接着点击chrome地址栏右侧的Proxy SwitchySharp插件图标，如下图所示，选择自动切换模式，这样使得我们上国内网站时不走代理，不影响国内网站的浏览。  
[![6](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-6.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-15-digitalocean-6.png)  　　
#4 完成　　
　　这样我们就通过shadowsocks使用自己的VPS进行科学上网了！最后提醒下当发现不能科学上网时，确定客户端已经设置完毕切且没有问题时，可以先将浏览器设置为全局代理模式，看是否可以工作，如果可以，那边是浏览器插件设置的问题，否则检查客户端中填写的信息与VPS上的配置信息是否一致。