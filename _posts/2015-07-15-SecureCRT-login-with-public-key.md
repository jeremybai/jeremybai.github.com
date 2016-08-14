---
layout: post
title: "DigitalOcean使用小记--SecureCRT以Public Key验证登录VPS"
description: ""
categories: 
- linux
- web
tags: [SecureCRT,DigitalOcean,VPS,Public Key]
---

　　在使用SecureCRT登陆VPS时，每次都要输入密码，其实除了使用密码之外还可以使用公钥来进行授权登录，这里说的公钥也就是之前博客中所讲到的SSH密钥对（SSH Key）对中的公钥，SSH密钥对可以让你方便的通过SSH登录到服务器，而无需输入密码，你不需要发送你的密码到网络中，SSH密钥对被认为是更加安全的方式。**SSH密钥对总是成双成对出现的，一把公钥，一把私钥。公钥(Public Key)可以自由的放在您所需要连接的SSH服务器上，而私钥(Private Key)必须稳妥的保管好。**所谓"公钥登录"，原理很简单，就是你将自己的公钥储存在远程主机上。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。这样子即可保证了整个登录过程的安全，也不会受到中间人攻击。  
　　接下来我们看下如何使用SecureCRT生成密钥对，打开SecureCRT之后,点击`工具－－创建公钥`，接下来的步骤如下面的图片所示：  
　　[![1](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-1.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-1.png)   
　　[![2](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-2.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-2.png)   
　　[![3](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-3.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-3.png)   
　　[![4](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-4.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-4.png)   　
　　[![5](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-5.png-BlogPic)](http://7fv9jl.com1.z0.glb.clouddn.com/2015-07-15-SecureCRT-login-with-public-key-5.png)  
　　创建完成之后我们的密钥对就存储在了最后一幅图中的路径下，`Identity`为私钥和`Identity.pub`为公钥，复制Identity.pub文件到服务器，放到home目录的.ssh子目录下，并执行命令：  
　　`# ssh-keygen -X -f Identity.pub > authorized_keys`  
　　这个命令的作用就是将公钥的内容复制到`authorized_keys`文件中。重新用SecureCRT连接一下试试，能够直接登录就好了！  
　　