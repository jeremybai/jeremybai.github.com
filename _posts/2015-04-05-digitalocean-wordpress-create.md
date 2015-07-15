---
layout: post
title: "DigitalOcean使用小记--CentOS7搭建WordPress"
description: ""
categories: 
- linux
- web
- WordPress
tags: [DigitalOcean,VPS,WordPress]
---


　　环境介绍：LAMP(Linux, Apache, MySQL和PHP) ，可以参考这个[链接](http://jeremybai.github.io/blog/2015/03/20/digitalocean-lamp/)。
##步骤1 创建MySQL数据库和用户
　　WordPress使用关系型数据库管理整个网站，我们使用的MariaDB（MySQL的一个分支）创建需要使用的数据库。使用MySQL的root用户登录：  

	mysql -u root -p
　　该命令输入完成之后会提示你输入密码（安装时设置的密码），验证之后进入MySQL的命令提示符。接着创建一个新的数据库，名称随便你取，我这里将它称为`wordpress`，别忘了命令后面的分号：  

	CREATE DATABASE wordpress;
　　接着创建用户`wordpressuser`，他的密码设置为`password`，你根据你的需求设置你喜欢的用户名和密码：  

	CREATE USER wordpressuser@localhost IDENTIFIED BY 'password';
　　接着我们需要将数据库和用户联系起来：  

	GRANT ALL PRIVILEGES ON wordpress.* TO wordpressuser@localhost IDENTIFIED BY 'password';
　　完成之后我们需要通知MySQL最近做出的改变：  

	FLUSH PRIVILEGES;
　　接着便可以退出了：  

	exit
　　此时又回到了你的ssh命令提示符了。  
##步骤2 安装WordPress
　　载安装WordPress之前我们需要安装一个php的模块：`php-gd`，它让WordPress可以修改图片的尺寸用于创建缩略图：  

	sudo yum install php-gd
　　重启Apache使得更改生效：  

	sudo service httpd restart
　　接下来我们就可以下载WordPress进行安装了，最新版本的WordPress的链接是不变的，因此通过下面的命令下载并解压：  

	cd ~
	wget http://wordpress.org/latest.tar.gz
	tar xzvf latest.tar.gz
　　接着将解压下来的文件拷贝至Apache的文档根目录（`/var/www/html/`）： 
 
	sudo cp -r ~/wordpress/* /var/www/html　　
　　
##步骤3 WordPress配置
　　安装包中给出了一个配置文件（wp-config-sample.php），将其复制改名为：`wp-config.php`，并且打开它进行修改：  

	cp ~/wordpress/wp-config-sample.php ~/wordpress/wp-config.php
	vi ~/wordpress/wp-config.php
　　将下面的DB_NAME，DB_USER和DB_PASSWORD修改为你之前设置的名称：  

	// ** MySQL settings - You can get this info from your web host ** //
	/** The name of the database for WordPress */
	define('DB_NAME', 'wordpress');

	/** MySQL database username */
	define('DB_USER', 'wordpressuser');

	/** MySQL database password */
	define('DB_PASSWORD', 'password');
　　修改完成之后便可以打开网页（example.com/wp-admin/install.php）填写好一些基本信息访问你的服务器地址就可以看到搭建好的WordPress了。