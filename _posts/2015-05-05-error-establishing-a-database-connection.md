---
layout: post
title: "DigitalOcean使用小记--Error Establishing a Database Connection错误"
description: ""
categories: 
- linux
- web
- WordPress
- 翻译
tags: [DigitalOcean,VPS,WordPress]
---

　　**原文地址**：[http://www.wpbeginner.com/wp-tutorials/how-to-fix-the-error-establishing-a-database-connection-in-wordpress/ ](http://www.wpbeginner.com/wp-tutorials/how-to-fix-the-error-establishing-a-database-connection-in-wordpress/)  

----------
　　
　　如果你一直在浏览网页，那么你一定看到过这个错误不止一次。有许多原有可能会导致Error Establishing a Database Connection这个错误。作为一个WordPress初学者，这个错误较长会让使用者手足无措，尤其是当你什么都没有做的情况下出现这个错误。我们在昨天在自己的网站上遇到了这个问题时花费了20几分钟解决它，在解决的过程中发现没有一篇文章包含了所有可能出现的原因。所以我们写了这篇文章将大部分可能的导致Error Establishing a Database Connection这个错误的原因都列出来。  
　　**PS：在你做任何的数据库改动之前，确保先进行备份。**  
##为什么会出现这个问题？
　　总而言之，这个错误出现的原因在于WOrdPress不能建立数据库连接。这个原因可能有很多，可能是数据库登陆验证错误或者被改动过了，可能是数据库服务器没有响应，也可能是数据库瘫痪了。在我们的经验中，最主要的原因可能是服务器错误，接下来让我们看看如何解决这个问题。  
##这个问题也出现在/wp-admin/上吗?
　　首先你需要确认这个问题发生在网站前端还是后端（wp-admin），如何前后端同时出现“Error establishing a database connection”这个错误，那么继续继续下一步。如果在wp-admin出现了其他类似“One or more database tables are unavailable. The database may need to be repaired”这种错误，那么你需要修复你的数据库。  
　　在你的wp-config.php文件中增加如下的配置： 

	define('WP_ALLOW_REPAIR', true);
　　增加配置之后打开下面这个页面查看设置：  

	http://www.yoursite.com/wp-admin/maint/repair.php
　　[![1](http://7fv9jl.com1.z0.glb.clouddn.com/2015-05-05-error-establishing-a-database-connection-1.gif) ](http://7fv9jl.com1.z0.glb.clouddn.com/2015-05-05-error-establishing-a-database-connection-1.gif)
　　打开这个网页进行修复是不需要用户进登录的，因为在修复一个崩溃的数据库时用户很有可能不能登录。所以当你修复优化完数据库之后却好将这行配置从wp-config.php中移除。 
　　如何修复之后还有问题或者在修复过程中出现问题那么你还需要继续阅读查看其它办法。  
##检查WP-Config文件 
　　WP-Config.php可能是你在WordPress安装过程中遇到的最重要的配置文件。连接数据库的相关配置都在这个文件中，如果你修改过数据库root密码或者数据库用户密码，那么你需要在这个文件中也需要进行相应的修改。第一件事你需要检查wp-config.php中的配置是否一致。  

	define('DB_NAME', 'database-name');
	define('DB_USER', 'database-username');
	define('DB_PASSWORD', 'database-password');
	define('DB_HOST', 'localhost');
　　记住DB_Host的值不一定是localhost，取决于你所用的主机，像常用的一些主机如HostGator, BlueHost, Site5使用localhost，你也可以在[这里](http://www.wpbeginner.com/wp-tutorials/useful-wordpress-configuration-tricks-that-you-may-not-know/)查看其他的主机值。  
　　有些人建议将localhost设置为IP，这种情况在[本地服务器上搭建WOrdPress](http://www.wpbeginner.com/wp-tutorials/installing-wordpress-on-a-local-server-environment/)时是很常见的。举例来说在MAMP上，将DB_Host 设置为IP是可以解决问题的。  

	define('DB_HOST', '127.0.0.1:8889');
　　在线的虚拟主机服务的IP是会变化的，所以当你的IP改变时记得修改。  
　　当你检查完所有的文件配置是正确的（拼写也要检查），那么问题应该就是出在服务器端了。  
##检查Web Host（MySQL Server）
　　当你的网站连接太多时也会看到这个错误，可能有是由于你的主机服务器不能处理负载（尤其是当你使用共享主机的时候），你的网站会变得很慢而且有的用户甚至会出现这个错误。所以最好的方法就是联系你的主机提供商咨询下你的MySQL服务器是否响应。 
　　如果想自己测试你的MySQL服务器是否运行，可以测试这个服务器上的其他网站是否存在这个问题，如果其他网站也出现这个问题，那么有可能就是你的MySQL服务器出了问题；如果这个服务器上没有其他网站可以测试，打开cPanel并且打开phpMyAdmin连接数据库，如果你能连接上的话那你就需要验证你的数据库用户是否拥有足够的权限。创建一个新的文件叫做testconnection.php并且将下列代码粘贴上去：  

	<?php
	$link = mysql_connect('localhost', 'root', 'password');
	if (!$link) {
	die('Could not connect: ' . mysql_error());
	}
	echo 'Connected successfully';
	mysql_close($link);
	?>
　　确保将用户名和密码替换掉，如果连接成功，表明你拥有足够的权限，错误发生在其他地方，回头检查wp-config文件确保所有选项正确（重新检查拼写）。  
　　如果使用phpMyAdmin无法连接到服务器，那么应该是你的服务器出了问题，但是这并不意味着是你的MySQL服务器停止服务，有可能是你没有足够的权限。  
　　在我们的情况中，MySQL服务器是在运行的，服务器上所有的其他网站都是在正常工作的除了WPBeginner。当我们想要登录phpMyAdmin时出现了这个错误： 
 
	#1045 – Access denied for user ‘foo’@’%’ (using password: YES)
　　我们打电话到HostGator，他们迅速帮我们找到了错误。他们重启了我们的权限。虽然不知道为什么会发生这个错误，但是很明显这个就是原因。重新恢复了权限之后我们的网站就恢复了。  
　　如果你在连接phpMyAdmin或者打开testconnection.php页面时出现权限不够的错误，你需要联系你的主机进行修复。  
##其他解决方法
　　上面的情况对你来说可能不适用，在操作之前一定要确保你们有足够的备份。  
　　[Deepak Mittal](http://www.absolutelytech.com/2010/06/19/solvederror-establishing-a-database-connection-in-wordpress/)说他的客户端也出现了这个错误需要修复数据库，修复完之后错误并没有消失。他试过许多种方法最后发现是因为网站url的问题。很明显是由于修改导致的持续的错误。他在phpMyAdmin运行SQL查询语句：  

	UPDATE wp_options SET option_value='YOUR_SITE_URL' WHERE option_name='siteurl'
　　一定要将YOUR_SITE_URL 替换为实际的url，比如说**http://www.wpbeginner.com**，[如果你修改了默认的WOrdPress数据库前缀](http://www.wpbeginner.com/wp-tutorials/how-to-change-the-wordpress-database-prefix-to-improve-security/)的话`wp_options`也会不同。  
　　这个方法帮他和其他一些在他文章后面评论的恩解决了问题。  
　　Sachinum说他可以通过testconnection.php连接到数据库，所以他将wp-config.php用户修改为root用户。修改完之后WordPress便可以正常工作，然后他又将配置改回去WordPress还是可以正常工作的，得出来的结论是这是个错误。  
　　Cutewonders建议移除wp_options表中active_plugins的内容，编辑recently_edited的内容。似乎也解决了问题，可以参考[这里](http://wordpress.org/support/topic/error-establishing-a-database-connection-351#post-2733573)。  
　　还有些其他用户重新上传了WordPress的最新副本之后这个问题就解决了。  
　　这个错误确实很奇怪，如果你有什么方法对你有用的，希望你可以告诉我们让我们加入到这里，这样其他用户就不会花费那么多时间去找解决方法了。  