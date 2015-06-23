---
layout: post
title: "DigitalOcean使用小记--如何在CentOS上搭建LAMP环境"
description: ""
categories: 
- linux
- web
- 翻译
tags: [DigitalOcean,VPS]
---


　　原文地址：[How To Install Linux, Apache, MySQL, PHP (LAMP) stack On CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-centos-7)。

----------
[![1](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-20-digitalocean-lamp-1.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-20-digitalocean-lamp-1.png)  
　　LAMP环境是一系列用来运行动态网站或者网页应用的开源软件的集合，它是一个缩写词，代表安装Apache服务器的Linux操作系统，网站的数据存储在MySQL数据库中（这里使用MariaDB），网站内容是由PHP处理的。  
　　在这篇文章中，我们将在安装了CentOS 7的VPS上安装LAMP，CentOS是一个Linux的发行版本，也就是LAMP的第一个条件。
# 先决条件
　　在你开始这个教程之前，你需要在你的VPS中拥有一个独立的非root账户，你可以参考[这篇文章](https://www.digitalocean.com/community/articles/initial-server-setup-with-centos-7)完成第一步到第四步。  
# 步骤1——安装Apache
　　Apache服务器是目前最流行的服务器之一，因此用它来架设网站是个不错的注意。  
　　我们可以使用CentOS的包管理器`yum`来安装Apache。包管理器使得我们可以无痛安装CentOS维护的库中的大部分软件，更多yum的信息可以参考[这里](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-use-yum-repositories-on-a-centos-6-vps)。  
　　使用下面的命令来安装Apache。    

    sudo yum install httpd
　　因为这个操作需要root权限，我们需要使用`sudo`命令，它会要求你输入密码验证你的身份。  
　　等待执行完成之后你的网站服务器便安装成功了，安装完成之后启动Apache：

    sudo systemctl start httpd.service
　　启动之后你便可以在你本地的浏览器中通过你的VPS的IP进行验证了（如果你不知道你的VPS的ip的话下一节的开头会有介绍如何获取）。

    http://your_server_IP_address/
　　此时就会看到用来显示基本消息和进行测试的CentOS 7上的Apache默认页面，应该和下图类似。    
[![2](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-20-digitalocean-lamp-2.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-20-digitalocean-lamp-2.png)  
　　如果你看到这个页面的话说明你的配置是没有问题的，接着讲Apache设置为开机启动：

    sudo systemctl enable httpd.service
## 如何获取你的服务器公共IP地址
　　如果你不知道你的服务器的公共IP的话，有许多方法可以用来获取它，通常来说它是你通过SSH访问VPS时所使用的地址。    
　　通过命令行也可以获取，使用`iproute2`根据获取公共IP方法如下：

    ip addr show eth0 | grep inet | awk '{ print $2; }' | sed 's/\/.*$//'
　　这个命令会返回若干行结果，他们都是正确的地址，但是你的计算机可能只能够使用其中一个，挨个试一下呗。  
　　还有通过第三方告诉你你的服务器IP，你可以询问一个指定的服务器让它告诉你你的VPS IP是多少：

    curl http://icanhazip.com
　　无论通过哪一种方法，最终你得到了你的IP，通过IP访问你的服务器吧。  
# 步骤2——安装MySQL(MariaDB)
　　现在你的网站服务器已经开始运行，现在可以安装数据库MariaDB，MariaDB数据库管理系统是MySQL的一个分支，主要由开源社区维护。它将组织并提供对数据库的访问和我们的网站信息的存储。  
　　继续使用`yum`进行安装，这一次我们还需要安装其他一些有用的包帮助我们组件之间相互继续通行。

    sudo yum install mariadb-server mariadb
　　完成安装之后开始启动mariaDB。

    sudo systemctl start mariadb
　　现在mariaDB已经开始启动了，现在我们需要运行简单的脚本来移除一些危险的默认设置并且锁定数据库的访问权限，执行命令：

    sudo mysql_secure_installation
　　脚本运行时会提示你输入你的root密码（不是你的系统的root密码，是数据库的root密码），因为你是第一次安装，所以你没有，只需要按下回车，此时脚本会要求你输入新的root密码，继续执行：

    Enter current password for root (enter for none):
    OK, successfully used password, moving on...
    
    Setting the root password ensures that nobody can log into the MariaDB
    root user without the proper authorization.
    
    New password: password
    Re-enter new password: password
    Password updated successfully!
    Reloading privilege tables..
     ... Success!
　　大部分情况下你只需要直接按下回车设置为默认值即可，这样会删除一些样例用户和数据库，禁止远程的root登陆以及使得MySQL立即执行这些修改。  
　　最后还是像之前一样将MariaDB设置为开机启动：

    sudo systemctl enable mariadb.service
　　安装好数据库之后我们便可以继续了。
# 步骤4——安装PHP
　　PHP是我们安装的组件之一用来将代码显示为动态内容。它可以运行脚本，连接到我们的MySQL数据库来获取信息，并把处理的内容传输到我们的Web服务器进行显示。  
　　我们可以再次利用yum安装我们的PHP组件，包括PHP-MySQL包（使得通过PHP操作数据）。

    sudo yum install php php-mysql
　　安装过程应该是没有任何问题的，重新启动Apache Web服务器才能使用PHP：

    sudo systemctl restart httpd.service
## 安装PHP模块
　　为了提高PHP的功能，我们可以选择安装一些额外的模块。  
　　为了查看PHP可用的模块和库，输入命令：

    yum search php-
　　返回的结果是所有可以安装的可选组件，每个都有一个简短的描述： 

    php-bcmath.x86_64 : A module for PHP applications for using the bcmath library
    php-cli.x86_64 : Command-line interface for PHP
    php-common.x86_64 : Common files for PHP
    php-dba.x86_64 : A database abstraction layer module for PHP applications
    php-devel.x86_64 : Files needed for building PHP extensions
    php-embedded.x86_64 : PHP library for embedding in applications
    php-enchant.x86_64 : Enchant spelling extension for PHP applications
    php-fpm.x86_64 : PHP FastCGI Process Manager
    php-gd.x86_64 : A module for PHP applications for using the gd graphics library
    . . .
　　为了知道这些模块的具体信息，你可以上网搜索或者输入命令：

    yum info package_name
　　这样会增加`Description`字段显示更多的模块信息。  
　　例如，为了查找php-fpm模块的详细信息，我们可以这样：

    yum info php-fpm
　　该命令会返回类似以下的信息：

    . . .
    Summary 	: PHP FastCGI Process Manager
    URL 		: http://www.php.net/
    License 	: PHP and Zend and BSD
    Description : PHP-FPM (FastCGI Process Manager) is an alternative PHP FastCGI
    			: implementation with some additional features useful for sites of
    			: any size, especially busier sites.
　　如果经过研究，你决定你想安装一个软件包，你可以使用yum install命令就像我们之前安装其他软件一样。   
　　如果我们发现php-fpm模块是我们需要，我们可以输入命令：

    sudo yum install php-fpm
　　如果你想安装一个以上的模块，你可以列出所有模块并用一个空格隔开，继续`yum install`命令，就像这样：

    sudo yum install package1 package2 ...
　　此时，你的LAMP环境已经安装结束并配置完成，接下来还需要对PHP进行测试。
# 步骤4——在服务器上测试PHP
　　为了测试我们的PHP配置是否正确，我们可以创建一个非常简单的PHP脚本。  
　　我们将这个脚本称为info.php。为了让Apache找到该文件，它必须被保存到一个特定的目录，被称为“网站根目录”。  
　　在CentOS 7，该目录位于/var/WWW /HTML/。在这个位置创建该脚本： 

    sudo nano /var/www/html/info.php
　　这将打开一个空白的文件。我们想把下面的文字拷贝至该文件，下面这段代码是有效的PHP代码：

    <?php phpinfo(); ?>
　　拷贝完成之后保存退出。  
　　如果你正在运行一个防火墙，运行下面的命令来允许HTTP和HTTPS流量： 

    sudo firewall-cmd --permanent --zone=public --add-service=http 
    sudo firewall-cmd --permanent --zone=public --add-service=https
    sudo firewall-cmd --reload
　　现在打开这个页面我们便可以测试是否能正确显示我们的Web服务器上PHP脚本生成的内容了，输入网址：  

    http://your_server_IP_address/info.php
　　你应该看到这样一个页面：  
[![3](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-20-digitalocean-lamp-3.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-03-20-digitalocean-lamp-3.png)  
　　该页面给出了从PHP的角度显示你的服务器的相关信息，在调试或者确保你的设置是否成功的时候这一点是很有用的。   
　　如果该页面显示成功，那么恭喜你，你的PHP可以正常工作了。   
　　可能你想测试完成之后删除这个文件因为它会向未经授权的用户暴露你的服务器信息。可以通过下面的命令删除该脚本：  

    sudo rm /var/www/html/info.php
　　在你需要这些信息时可以再次创建这个页面。
# 结论
　　现在你有一个安装好的LAMP环境了，在它上面你可以选择你想做的，基本上来说它已经可以让你在你的服务器上安装绝大多数的网站和网络软件了。  

















