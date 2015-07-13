---
layout: post
title: "DigitalOcean使用小记--如何将wordpress置于独立文件夹"
description: ""
categories: 
- linux
- web
- WordPress
tags: [DigitalOcean,VPS,WordPress]
---


　　在使用wordpress的时候，很多人希望在根目录（比如 http://example.com）使用WordPress，但又不希望让WordPress将根目录弄乱。WordPress允许将它安装在一个单独的目录中，但是，你的网站必须要从根目录启动。从WordPress3.5开始,多站点用户也可以使用该功能。具体步骤如下：  
　　1 首先需要创建一个用语存放WordPress的目录，我将目录建在了`/var/www/html/wordpress`路径。  
　　2 打开你已有的WordPress设置页面，也就是你的仪表盘，在`常规－设置`选项中，将`WordPress地址（URL）`选项修改为你的WordPress核心文件的新位置，例如：`http://example.com/wordpress` 。站点地址（URL）选项还是你的站点根目录地址，例如：`http://example.com`。   
　　3 点击`保存更改`，此时应该会跳出错误信息，不用理它。  
　　4 将你站点根目录下的所有WordPress相关的文件移至`/var/www/html/wordpress`目录中（根据自己所建的目录修改改目录），再将新位置中的index.php文件和.htaccess（如果存在.htaccess文件的话）文件拷贝至根目录下。  
　　5 编辑index.php，将其中的

	require( dirname( __FILE__ ) . '/wp-blog-header.php' );
　　修改为

	require( dirname( __FILE__ ) . '/wordpress/wp-blog-header.php' );　　
　　修改完之后保存退出，在新的位置登陆，例如：`http://example.com/wordpress/wp-admin/`，在登陆时访问可能会比较慢，耐心等待几分钟应该就可以了。
　　
