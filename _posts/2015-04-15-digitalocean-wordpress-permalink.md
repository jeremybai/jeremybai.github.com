---
layout: post
title: "DigitalOcean使用小记--设置wordpress使用固定链接"
description: ""
categories: 
- linux
- web
- WordPress
tags: [DigitalOcean,VPS,WordPress]
---

　　固定链接是个人博客里的文章、分类以及其他页面的固定链接地址。通过固定链接可以链到你写的博客，你也可以将这个链接地址写在邮件里发给其他人看。如果博客的链接地址变来变去，会造成其他人通过之前的链接地址来浏览博客时出错，所以每篇博客的链接地址都应该固定，而且永久不改，这也是固定链接名字的由来。  
　　WordPress默认的链接类型是PATHINFO类型的，PATHINFO类型的链接格式一般如下:

	http://example.com/index.php/yyyy/mm/dd/post-name/
　　我希望可以直接变为下面这种格式：  

	http://example.com/year/month/day/post-name
　　通过mod_rewrite可以生成这种类型的链接格式。 
## WordPress设置 
　　首先打开WordPress的仪表盘，选择`设置－固定链接设置`，选中你喜欢的链接类型，当然也支持自定义，选择之后点击保存更改，这时WordPress会根据你的.htaccess文件是否可写给出提示：  

	如果您的.htaccess文件可写，我们即会自动帮您完成，但其目前不可写，所以以下是您需要加入您的.htaccess文件中的mod_rewrite规则。点击文本框并按CTRL + a来全选。
	<IfModule mod_rewrite.c>
	RewriteEngine On
	RewriteBase /
	RewriteRule ^index\.php$ - [L]
	RewriteCond %{REQUEST_FILENAME} !-f
	RewriteCond %{REQUEST_FILENAME} !-d
	RewriteRule . /index.php [L]
	</IfModule>
　　一般.htaccess文件是不可写的（如果你没有这个文件的话就新建这个文件：`touch .htaccess`），这时你需要将给出的代码手动加入到.htaccess文件中去。重启下httpd服务（`service httpd restart`），此时点开文章的链接发现找不到路径，不用慌，因为你还没有开启mod_rewrite模块。  
## Apache开启mod_rewrite　　
　　在使能mod_rewrite之前，你需要确认两件事，你是否安装了这个模块以及Apache是否加载了mod_rewrite模块。通过`httpd -V`命令确定Apache配置文件的位置： 
 
	[jeremybai@droplet1 ~]$ httpd -V
	Server version: Apache/2.4.6 (CentOS)
	Server built:   Mar 12 2015 15:07:19
	Server's Module Magic Number: 20120211:24
	Server loaded:  APR 1.4.8, APR-UTIL 1.5.2
	Compiled using: APR 1.4.8, APR-UTIL 1.5.2
	Architecture:   64-bit
	Server MPM:     prefork
	  threaded:     no
	    forked:     yes (variable process count)
	Server compiled with....
	 -D APR_HAS_SENDFILE
	 -D APR_HAS_MMAP
	 -D APR_HAVE_IPV6 (IPv4-mapped addresses enabled)
	 -D APR_USE_SYSVSEM_SERIALIZE
	 -D APR_USE_PTHREAD_SERIALIZE
	 -D SINGLE_LISTEN_UNSERIALIZED_ACCEPT
	 -D APR_HAS_OTHER_CHILD
	 -D AP_HAVE_RELIABLE_PIPED_LOGS
	 -D DYNAMIC_MODULE_LIMIT=256
	 -D HTTPD_ROOT="/etc/httpd"
	 -D SUEXEC_BIN="/usr/sbin/suexec"
	 -D DEFAULT_PIDLOG="/run/httpd/httpd.pid"
	 -D DEFAULT_SCOREBOARD="logs/apache_runtime_status"
	 -D DEFAULT_ERRORLOG="logs/error_log"
	 -D AP_TYPES_CONFIG_FILE="conf/mime.types"
	 -D SERVER_CONFIG_FILE="conf/httpd.conf"
　　其中`HTTPD_ROOT`和`SERVER_CONFIG_FILE`合起来便代表了配置文件的位置：   

	/etc/httpd/conf/httpd.conf
　　Apache模块的路径在： 
 
	/etc/httpd/modules
　　通过下面的命令查看是否存在mod_rewrite模块：
  
	ls /etc/httpd/modules | grep mod_rewrite
　　如果出现了下面的信息，代表该模块以及存在，你只需要加载它就可以了，如果没有显示任何消息，则需要重新编译Apache生成改模块。  

	[jeremybai@droplet1 ~]$ ls /etc/httpd/modules | grep mod_rewrite
	mod_rewrite.so
　　再通过下面的命令来确认mod_rewrite是否被加载或者使能： 
 
	grep -i LoadModule /etc/httpd/conf/httpd.conf | grep rewrite
　　如果出现了下面的信息，表明mod_rewrite模块已经被使能了。

	[jeremybai@droplet1 ~]$ grep -i LoadModule /etc/httpd/conf/httpd.conf | grep rewrite
	LoadModule rewrite_module modules/mod_rewrite.so
　　如果你看到了下面的信息，将其注释给取消掉：    

	#LoadModule rewrite_module modules/mod_rewrite.so
　　如果你什么都没看到，将这一行代码加到`httpd.conf`文件中去：  

	LoadModule rewrite_module modules/mod_rewrite.so
　　现在，mod_rewrite已经存在并且被加载，为了通过mod_rewrite达到使用.htaccess进行url重写的目的，需要覆盖Apache原来的全局选项，使用命令：   

	grep -i AllowOverride /etc/httpd/conf/httpd.conf
　　将会输出：  

	[jeremybai@droplet1 ~]$ grep -i AllowOverride /etc/httpd/conf/httpd.conf
    AllowOverride None
　　你需要将所有的`AllowOverride None`改为`AllowOverride All`。修改完成之后，重新点开你的文章的链接你就会发现你的文章的url已经改变了！
　　
　　
　　
　　