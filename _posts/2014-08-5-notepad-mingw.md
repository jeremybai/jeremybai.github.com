---
layout: post
title: "如何在notepad++中调用MinGW编译运行程序"
description: ""
categories: 
- C
tags: [notepad++,MinGW]
---
{% include JB/setup %}
[![1](http://github-blog.qiniudn.com/2014-08-5-notepad-mingw-1.png-BlogPic) ](http://github-blog.qiniudn.com/2014-08-5-notepad-mingw-1.png)   
　　平时在Windows写一些不大的程序的时候，喜欢使用notepad++来写，原因就是一个字：快！可是写完之后编译运行还是要放到一些IDE中去运行，电脑配置不高，打开个环境还要个几分钟，于是就想着能不能在windows上调用GCC，搜了下，还真有，GNU的大神们已经将GCC编译器和GNU Binutils（一整套的编程语言工具程序，其中包括as（汇编器），ld（链接器）等等命令）移植到Windows平台下，叫做MinGW（Minimalist GNU for Windows），包括一系列头文件（Win32API）、库和可执行文件。GCC支持的语言大多在MinGW也受支持，其中包括C、C++、Objective-C、Fortran及Ada。对于C语言之外的语言，MinGW使用标准的GNU运行库，如C++使用GNU libstdc++。但是MinGW使用Windows中的C运行库。因此用MinGW开发的程序不需要额外的第三方DLL支持就可以直接在Windows下运行，而且也不一定必须遵从GPL许可证。这同时造成了MinGW开发的程序只能使用Win32API和跨平台的第三方库，而缺少POSIX支持，大多数GNU软件无法在不修改源代码的情况下用MinGW编译[1]。下面介绍下如何使用。

## 1 下载 ##
　　实现需要下载MinGW，你可以在[官网](http://www.mingw.org/)下载安装程序，不过安装程序都是在线安装的，所以你需要等待比较久的时间，我将我自己已经安装好的MinGW打了个包，放在[网盘（点击下载）](http://pan.baidu.com/s/1pJCzs1P)上，你可以直接下载下来解压到任意的地址,win7系统32位和64位都可以使用，我这里以C:\Program Files\MinGW为例。
## 2 添加环境变量 ##
　　右击计算机-->属性-->高级系统设置-->环境变量，在用户变量中添加如下几项，如果变量名已经存在就在变量值后面加上分号继续添加。如图所示：
[![2](http://github-blog.qiniudn.com/2014-08-5-notepad-mingw-2.png-BlogPic)  ](http://github-blog.qiniudn.com/2014-08-5-notepad-mingw-2.png)  
{% highlight c++ %}
变量名：CPLUS_INCLUDE_PATH
变量值：C:\Program Files\MinGW\include\c++\3.4.2;C:\Program Files\MinGW\include\c++\3.4.2\mingw32;C:\Program Files\MinGW\include\c++\3.4.2\backward;C:\Program Files\MinGW\include;
变量名：path
变量值：C:\Program Files\MinGW\bin;
变量名：C_INCLUDE_PATH
变量值：C:\Program Files\MinGW\include;
变量名：LIBRARY_PATH
变量值：C:\Program Files\MinGW\lib;
{% endhighlight %}
　　这些变量主要是指定你运行的gcc/g++/gdb等等命令的路径、可能需要用的库的路径和包含的头文件的路径。我这里添加的路径是针对下载我的MInGW包的同学，如果你是在线安装最新版本的MinGW的话，对应的路径可能会不太一样，所以要现在安装目录中查找对应的目录，将路径加上即可。
## 3 测试 ##
　　为了测试MinGW的安装，打开命令行，输入`gcc -v`，如果出现了gcc的版本，就表明已经安装成功了，
[![3](http://github-blog.qiniudn.com/2014-08-5-notepad-mingw-3.png-BlogPic)  ](http://github-blog.qiniudn.com/2014-08-5-notepad-mingw-3.png)  
## 4 增加notepad++宏 ##
　　接下来要做的就是在notepad++中调用MinGW了，打开notepad++，写上一个最简单的HelloWorld.c程序，
点击菜单栏的《运行》按钮，输入命令：
{% highlight c++ %}
cmd /k g++ -o $(CURRENT_DIRECTORY)\$(NAME_PART).exe "$(FULL_CURRENT_PATH)" & PAUSE & EXIT
{% endhighlight %}
点击运行，此时会出现一个命令行，如果你的代码有问题，会出现错误提示，如果没有错误，就会出现“请按任意键继续”的提示。此时在点击菜单栏的《运行》按钮，输入下面的命令：
{% highlight c++ %}
cmd /k "$(CURRENT_DIRECTORY)\$(NAME_PART)" & PAUSE & EXIT
{% endhighlight %}
点击运行，如图所示此时出现的命令行就会显示你的程序的运行结果。
[![4](http://github-blog.qiniudn.com/2014-08-5-notepad-mingw-4.png-BlogPic) ](http://github-blog.qiniudn.com/2014-08-5-notepad-mingw-4.png)  
你可以将这两个命令保存为快捷键（点击菜单栏的《运行》按钮，输入完命令后点击保存就可以了），这样运行起来比较方便。到此，notepad++中调用GCC编译程序就完成了。
### 参考文献
　　[1] [http://zh.wikipedia.org/wiki/MinGW ](http://zh.wikipedia.org/wiki/MinGW)

