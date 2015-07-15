---
layout: post
title: "DigitalOcean使用小记--WordPress使用ftp及问题"
description: ""
categories: 
- linux
- web
tags: [DigitalOcean,VPS]
---


　　在使用WordPress时需要通过FTP登录至服务器安装主题或者插件，vsftpd是一个UNIX类操作系统上运行的服务器，是“very secure FTP daemon”的缩写，这里简要介绍在CentOS 7下安装vsftpd以及简单的配置登陆过程。  
#1 安装vsftpd
　　使用下面的命令查看机器上是否已经安装了vsftpd：  

	rpm -qa | grep vsftpd
     
　　如果没有安装，使用下面的命令安装：  

	yum -y install vsftpd 	
	
　　设置开机启动： 
	 
	chkconfig vsftpd on


#2 修改配置

　　修改配置文件，配置文件位于`/etc/vsftpd/vsftpd.conf`，  

	anonymous_enable=NO //设定不允许匿名访问
	local_enable=YES //设定本地用户可以访问。注：如使用虚拟宿主用户，在该项目设定为NO的情况下所有虚拟用户将无法访问
	chroot_list_enable=YES //使用户不能离开主目录
	ascii_upload_enable=YES
	ascii_download_enable=YES //设定支持ASCII模式的上传和下载功能
	pam_service_name=vsftpd //PAM认证文件名。PAM将根据/etc/pam.d/vsftpd进行认证
　　修改完配置之后重启vsftpd服务：

	service vsftpd restart
　　便可以通过`ftp example.com`登录到vps的ftp服务器上了。还有些其他的命令：  
  
	# 启动vsftpd服务：
	service vsftpd start
	# 停止vsftpd服务：
	service vsftpd stop
	
##问题1: 以root权限登陆vsftpd
　　如果你想以root权限登陆vsftp的话，还需要做如下设置，因为vsftpd默认是不允许root用户登录的：  

	1、注释/etc/vsftpd.ftpusers中的root
	2、注释/etc/vsftpd.user_list中的root
	3、更改/etc/vsftpd/vsftp.conf中的userlist_enabel=NO
	4、service vsftpd restart
##问题2: WordPress更新主题时ftp连接不上
　　问题描述：

	要执行请求的操作，WordPress需要访问您网页服务器的权限。 请输入您的FTP登录凭据以继续。 如果您忘记了您的登录凭据（如用户名、密码），请联系您的网站托管商。
　　解决方法：在wp-config.php里加入下面代码:

	define("FS_METHOD", "direct");
	define("FS_CHMOD_DIR", 0777);
	define("FS_CHMOD_FILE", 0777);
##问题3: WordPress安装插件时提示 “无法创建目录”
　　WordPress安装插件时提示“无法创建目录”，这个是由于目录的权限设置导致的，一般情况下可以直接给目录/wp-content目录增加777权限，目录地址改为你的wordpress安装地址下的wp-content文件夹：  

	chmod -R 777 ./wordpress/wp-content
　　改完权限之后就可以安装插件或者主题了。