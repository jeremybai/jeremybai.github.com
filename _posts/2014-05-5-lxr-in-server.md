---
layout: post
title: "Redhat非root权限源码编译安装lxr过程"
description: ""
categories: 
- 翻译
tags: [VGA,嵌入式]
---
{% include JB/setup %}

　　最近被布置了个任务，在服务器上搭建lxr用来查看项目的源码，整整一个星期一直在折腾这个，导致我夜里做梦还梦见自己在搭环境，过程中遇到了各种各样的问题，记录下来给自己也给需要的同学看看，避免以后走弯路。

----------

　　首先介绍下lxr到底是个啥？其实我自己之前也不知道，搜了下，lxr是“the Linux Cross Referencer”的缩写，中间的“X”形象地代表了“Cross”，看名字可以大概看的出来它是干嘛的，用来查看linux源码的交叉引用的。最初只是被用于查看Linux的源代码，后来被发现也可以应用于其他的软件工程中。官方网站：[http://lxr.sourceforge.net/en/index.shtml](http://lxr.sourceforge.net/en/index.shtml)，源码下载地址：[http://sourceforge.net/projects/lxr/](http://sourceforge.net/projects/lxr/)

　　平时自己在看代码的时候要不就是直接在IDE中看，跳转还是蛮方便的，在服务器上就是用vim看，这个就比较蛋疼了，虽然加上ctags可能方便不少，但是总觉得没有鼠标点来点去的方便（原谅我这一生不羁放纵喜欢鼠标点击跳转=。=！，哎，都是微软给“惯”的）。lxr运行用户通过浏览器查看代码，并且所有变量、常量、函数都以超链接的形式给出，十分方便查阅。比如你阅读源码，发现函数不知道是如何以及在哪里定义的，这时候你只要点击这个函数,lxr将返回此函数的定义、实现以及各次引用是在什么文件的哪一行，同样的，这些信息也是超连接，点击将直接跳转到相应的文件相应的行。另外lxr还提供标识符搜索、文件搜索等功能，同时结合glimpse或者swish-e（后面会提到）提供对所有的源码文件进行全文检索，甚至包括注释。

　　介绍完功能，我们便开始尝试搭建环境，我自己当时在服务器上搭建之前，现在自己的虚拟机上搭建了一下，虚拟机装的是ubuntu，不行的时候就su root，所以有些软件在装的时候直接使用的apt-get的安装的，2天就搞定了，可是在服务器上是不允许你拥有root权限的，所以所有的环境搭建都是在普通账号下搭建的，这样就不可以使用yum这样的包安装了，因为它需要root，所有的环境搭建都需要自己下载源码编译安装，下面就开始lxr的的坑爹之旅。

##1 环境部署
　　前期准备需要：
###1.1 Perl解释器
　　一般系统都会默认安装的，perl的版本必须大于等于5.10，使用perl –v命令查看版本（实在达不到就安装LXR 0.11.1，这个版本是最后兼容的），由于服务器上默认安装的perl版本比较低，所以只能自己再装一个。Perl 5.18.2安装有问题，make install时需要root权限，我就使用了5.14.4，从官网下载tar包解压缩进入目录运行：

	CFLAGS='-m64 -mtune=nocona' ./Configure -des -A ccflags=-fPIC -Dprefix=/home/hao123/lxr_env/perl
	make   
	make install

　　这里的配置命令加了一些配置，是针对64位系统的，本来就直接加了-Dprefix，但是在安装mod_perl时出了问题，查找原因时发现是perl在安装到64位系统时要加其他几个参数，然后下面会讲到。安装完成之后记得查看当前perl的版本，如果还是之前的版本的话将perl的安装路径添加到系统路径，也就是当前账号的home路径下的.bashrc文件

	export PATH="<你的安装路径>/perl/bin:$PATH"
　　放在前面保证不被root的perl路径覆盖掉。  
　　ps：在源码编译安装时，最好使用--prefix指定安装的位置，这样更新或者删除时比较方便，否则默认安装可能会将代码发布到不同的文件夹导致维护起来不方便。
###1.2 exuberant ctags
　　lxr中也是用到了ctags这个代码跳转利器，命令ctags --version查看版本，没有的话下载源码安装，这个就比较简单了，简单指定下安装路径就可以了，我使用的是5.5.4。
###1.3 关系型数据库
　　数据库用于存放生成的代码索引，MySQL 4.x/5.x, Oracle, Postgresql 还有SQLite 都可以。我使用的是mysql5.6.17。  
　　写到这里我不得不吐槽下，我在安装mysql时，发现服务器上之前有人安装过，心中窃喜，可以直接用了，可是问了一圈居然没人知道mysql的root密码，网上找了找，忘记mysql的密码找回，通过跳过授权表启动mysql，然后在里面修改密码，试了试果然可以，此时一个朋友跟我聊天时突然说一般自己安装mysql时初始root密码一般没有设置，也就是空，试了下，果然mysql -u root -p直接回车就可以进去了！too young too simple啊。。。事儿还没完，进去了以为就可以用了，准备修改下root密码，发现找不到mysql数据库中的user表，show databases查看了下，发现里面的数据库都被删了，连mysql这个数据库都被删了。给跪！再给恢复这个数据库还不如自己重新装一下，于是又下载源码重新装了。  
　　回归正题，编译mysql需要安装cmake，没有还是自己装！=。=！然后使用cmake命令来编译，如果第一次编译失败或者有问题时删除mysql目录下的CMakeCache.txt重新编译即可，网上找了找配置的命令：

	cmake -DCMAKE_INSTALL_PREFIX=/home/hao123/lxr_env/mysql \
	-DSYSCONFDIR=/path/to/mysql/etc \
	-DMYSQL_DATADIR=/path/to/mysql/data\
	-DMYSQL_TCP_PORT=3306 \
	-DMYSQL_USER=mysql \
	-DEXTRA_CHARSETS=all \
	-DWITH_READLINE=1 \
	-DWITH_SSL=system \
	-DWITH_EMBEDDED_SERVER=1 \
	-DENABLED_LOCAL_INFILE=1 \
	-DWITH_INNOBASE_STORAGE_ENGINE=1 \
	-DWITHOUT_PARTITION_STORAGE_ENGINE=1 \ 
	-DENABLE_DOWNLOADS=1

　　cmake时报错：

	CMake Error: Problem with tar_extract_all(): Invalid argument
	CMake Error: Problem extracting tar: /usr/local/src/mysql-5.6.13/source_downloads/gmock-1.6.0.zip

　　报错的原因显示是tar解包gmock-1.6.0.zip出错了。怎么能用tar去解包zip压缩包呢？另外网上搜索了以下，gmock-1.6.0.zip是google的c++mock框架，从mysql 5.6开始支持。cmake参数中设置了DENABLE_DOWNLOADS=1且服务器能连接Internet的话，就会自动下载。上面报错信息可知，cmake时gmock-1.6.0.zip自动下载到了.../mysql-5.6.17/source_downloads/目录下，尝试手动编译安装gmock，然后再cmake：

	1 cd到mysql5.6源码文件夹下的source_downloads文件夹
	2 cd /path/to/mysql-5.6.17/source_downloads/
	3 unzip gmock-1.6.0.zip
	4 cd gmock-1.6.0
	5 ./configure
	6 make
　　没有解决，提示错误不能unzip，怎么办呢？于是我决定只保留路径相关的配置，把其他的参数全部去掉，变成:

	cmake -DCMAKE_INSTALL_PREFIX=/path/to/mysql \
	-DSYSCONFDIR=/path/to/mysql \
	-DMYSQL_DATADIR=/path/to/mysql/data \
	-DMYSQL_TCP_PORT=3306 \
	-DMYSQL_USER=mysql  \
	-DMYSQL_UNIX_ADDR=/path/to/mysql/mysql.sock
　　配置完成之后make  &&   make test   &&    make install就可以了。然后创建数据库

	/path/to/mysql/scripts/mysql_install_db   --user=mysql   
　　上述建库语句将根据my.cnf里设置的数据文件目录和日志文件目录，生成相应的数据文件和日志文件，并创建系统数据库（如mysql,test,information_schema,performance_schema)，接着启动MySQL

	/path/to/mysql/support-files/mysql.server start  
　　启动成功后，就可以使用命令mysql -u root -p以root用户登录（默认的root用户没有密码，记得把路径带入到.bashrc中）。

###1.4 webserver
　　推荐使用Apache httpd加上mod_perl，lighttpd 也可以。从 LXR 2.0开始, Cherokee, Nginx 和thttpd也被支持了。我使用的是apache+mod_perl。
　　PS：mod_perl是一个apache 模块，它巧妙的将 perl 程序语言封装在 Apache web 服务器内。在 mod_perl 下，CGI 脚本比平常运行快50倍。另外，可将数据库与web服务器集成在一起，用Perl编写Apache模块，在Apache的配置文件里面插入 Perl 代码，甚至以 server-side include 方式使用 Perl。在 mod_perl 下，Apache不仅仅是一个web 服务器，而变成了一个功能完善的程序平台。  
　　因为apache的配置依赖于**apr**、**apr-util**和**pcre**，所以首先安装这三个模块（否则在./configure的时候会报错）
####（1）安装apr 1.5.1
	wget  http://mirror.bit.edu.cn/apache//apr/apr-1.5.1.tar.gz
	tar  –zxf  apr-1.5.1.tar.gz
	cd  apr-1.5.1
	./configure  --prefix=/usr/local/apr 
	make
	make install
####（2）安装apr-util 1.5.3
	wget  http://mirror.bit.edu.cn/apache//apr/apr-util-1.5.3.tar.gz
	tar  –zxf  apr-util-1.5.3.tar.gz
	cd  apr-util-1.5.3
	./configure  --prefix=/usr/local/apr-util  --with-apr=/usr/local/apr/bin/apr-1-config 
	make
	make install
####（3）安装pcre 8.34
	wget  http://softlayer-dal.dl.sourceforge.net/project/pcre/pcre/8.34/pcre-8.34.tar.gz
	tar  –zxf  pcre-8.34.tar.gz
	cd  pcre-8.34
	./configure –prefix=/usr/local/pcre --with-apr=/usr/local/apr/bin/apr-1-config 
	make
	make install
####（4）安装Apache 2.2.27
　　Apache不能安装2.4的，因为mod\_perl还不支持，所以安装mod\_perl的时候会编译报错，错误信息参考这里[http://stackoverflow.com/questions/21629110/error-while-installing-mod-perl2](http://stackoverflow.com/questions/21629110/error-while-installing-mod-perl2)，所以最好装Apache2.2版本，因为这个问题卡了好久。

	wget  http://apache.fayea.com/apache-mirror//httpd/httpd-2.2.27.tar.gz
	tar  –zxf  httpd-2.2.27.tar.gz
	cd  httpd-2.2.27
	./configure --prefix=/usr/local/apache --enable-so --enable-mods-shared=all --enable-rewrite --enable-expires --enable-cache --with-apr=/usr/local/apr/bin/apr-1-config  --with-apr-util=/usr/local/apr-util/bin/apu-1-config  --with-pcre=/usr/local/pcre/bin/pcre-config
	make
	make install
####（5）安装mode_perl 2.0.8
　　在安装mod_perl的时候,配置时出现错误：

	/usr/bin/ld: /usr/local/lib/perl5/5.10.1/x86_64-linux/CORE/libperl.a(op.o): \
	  relocation R_X86_64_32S against `PL_sv_yes' can not be used when making a shared \
	                  object; recompile with -fPIC
	  /usr/local/lib/perl5/5.10.1/x86_64-linux/CORE/libperl.a: could not read symbols: Bad \
	                  value
　　这个错误是因为我的系统是64位的，如果之前安装perl时没加上几个参数（安装perl时讲到过），这时候就需要重新编译安装perl。装完重新配置mod_perl，配置时注意MP_APU_CONFIG=/usr/local/apr-util/bin/apu-1-config这个配置，这个是指定apr-util的路径，不加的话会提示错误：  
![1](http://github-blog.qiniudn.com/2014-05-5-lxr-in-server-1.png-BlogPic)

　　最后的安装如下过程如下：  

	wget  http://apache.org/dist/perl/mod_perl-2.0.8.tar.gz
	tar  –zxf  mod_perl-2.0.8.tar.gz
	cd  mod_perl-2.0.8
	perl Makefile.PL MP_APXS=/usr/local/apache/bin/apxs MP_APR_CONFIG=/usr/local/apr/bin/apr-1-config MP_APU_CONFIG=/usr/local/apr-util/bin/apu-1-config
	make
	make install
　　完成之后再把LoadModule perl\_module modules/mod\_perl.so这句加到/usr/local/apache/conf/ httpd.conf 中，重启apache（/path/to/apache/bin/apachectl restart），输入命令：/usr/local/apache/bin/apachectl -t -D DUMP\_MODULES测试是否包含mod\_perl，出现mod\_perl表明成功。  
　　apache安装完打开浏览器输入服务器的地址以及端口号，会显示it works页面，如果不能打开it works页面：原因：默认安装完使用的是80端口，0~1024是系统使用的端口，所以使用需要root权限，解决方法：  

	（1）使用root权限  
	（2）将端口改为大于1024的端口号

###1.5 文本检索工具
　　用于代码索引，Glimpse 或者Swish-e（2.1或者更高）都行，区别在于Glimpse在非商业用途是免费的，Swish-e是完全GPL许可的。功能上来说的话，Glimpse 可以定位到目标出现在具体哪一行，而且一个文件中有多个目标出现时可以区别；Swish-e能定位到目标出现的文件，把一个文件中出现的目标都合并了，但是，它可以支持CVS索引。我使用的是swish-e 2.4.7，下载源码指定路径安装，并且添加到系统路径。
###1.6 Perl的DBI和使用的数据库的DBD
　　DBI是Perl对于很多数据库的一个通用接口，DBD是相应数据库的驱动程序，我这边使用的MySQL，所以使用的应该就是DBD::mysql。指定目录安装DBI没有问题，接着安装DBD时就会出现错误。

	Can't locate DBI/DBD.pm in @INC。。。。
　　应该是没有找到DBI.pm，指定路径安装导致它不在查找的路径中（perl -e 'print join "\n",@INC'查看perl搜索的路径）。
　　解决方法：DBI和DBD::mysql都不指定目录安装即可。直接perl MakeFile.pl配置然后make && make install。

###1.7 Perl 的File::MMagic 模块
　　File::MMagic模块的安装与DBI和DBD类似，也是直接不指定路径安装，这样会把它安装到默认的perl存放模块的路径。

　　这样就完成了基本的环境搭建，其实这一步完成了，其他也就没什么了，剩下的都比较简单了。
##配置LXR
　　我使用的是xlr1.1.0，首先从官网那个上下载源码，解压到自己的home目录下，将文件夹改名为lxr，然后进入目录，运行命令

	./genxref --checkonly
　　检查下运行环境
![2](http://github-blog.qiniudn.com/2014-05-5-lxr-in-server-2.png-BlogPic)  

　　glimpse和swish-e只要有一项安装即可。接下来运行命令

	./scripts/configure-lxr.pl -vv 
　　这个目录是用于lxr配置文件以及数据库配置文件的生成，因为这个脚本时交互的，所以这里我就不列举过程了，可以参考[http://lxr.sourceforge.net/en/1-0-InstallSteps/1-0-install3config.shtml?](http://lxr.sourceforge.net/en/1-0-InstallSteps/1-0-install3config.shtml?)官网的说明，写的比较详细，初次搭建时有默认值就选默认值，没有的安装教程指示即可。完成之后配置文件也就生成好了，在./custom.d目录下，运行初始化数据库配置脚本：

	./custom.d/initdb.sh
　　此时就会在mysql中创建用户lxr以及相关的表，再把lxr.conf拷贝到当前目录：

	cp custom.d/lxr.conf .
　　拷贝完成之后运行：

	./genxref --url=<服务器的地址+端口号>/lxr --version=v1
　　genxref脚本就是用来对你指定的源码位置进行建立索引的，所以如果你的代码量很大的话，需要的实现可能会比较长，我第一次使用的linux-2.6.32内核源码进行编译，用了1个小时40分钟左右。其实可以自己在你指定的源码目录下建个helloworld程序用来测试一下，注意，在执行`./scripts/configure-lxr.pl -vv`命令时会让你输入源码的路径，这个路径如果没有，你要手动创建，比入我输入的路径是/path/to/lxr_src，那么在这个路径下面，你需要自己建文件夹，叫v1，然后把你的源码放进去，--url参数指定的是'host_names'加上'virtroot'，这两个参数是在lxr.conf中定义的，你打开lxr.conf文件，找到host_names这一项，添加你的服务器地址进去：  
![3](http://github-blog.qiniudn.com/2014-05-5-lxr-in-server-3.png-BlogPic)
　　--version指定的是需要建立索引的版本号，我这里只建立了一个v1文件夹所以指定v1版本。最后再将custom.d/apache-lxrserver.conf这个文件拷贝到/path/to/apache/conf目录下，然后在其中的httpd.conf文件的最后加上Include conf/apache-lxrserver.conf。然后访问<服务器地址+端口号>/lxr/source/就可以看到了。
