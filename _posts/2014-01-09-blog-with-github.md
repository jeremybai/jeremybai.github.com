---
layout: post
title: "使用Github搭建自己的博客"
description: ""
category:  工具使用
tags: [Jekyll]
---
{% include JB/setup %}
##1 搭建环境
操作系统：Windows 7旗舰版64位

##2 准备工作
确保你的电脑上已经安装了[Git](http://msysgit.github.io/ "msysGit"),[Ruby](http://rubyinstaller.org/downloads/ "Ruby Installer")和Jekyll，并且你已经拥有了一个github账户。   
###注意：
在安装ruby的时候选择Ruby1.9.3这个版本，因为这个版本比较稳定并且提供给了比较全的包的支持，在安装过程中将是否添加系统路径打上勾，装完Ruby再把DEVELOPMENT KIT也装下，记得选择与Ruby1.9.3相匹配的开发包下载。
安装完之后打开cmd，输入命令：

    ruby -v
如果能显示版本号，就说明安装成功。

安装Jekyll：在git bash中运行命令：

    gem install jekyll
耐心等待一段时间就可以了。安装成功后输入命令：

    jekyll -v
和上面一样，显示版本号就表明安装成功了。这个时候，我们的准备工作已经完成，接下来我们会在三分钟制备搭建一个bootstrap风格的个人博客。
##3 搭建博客
###3.1 创建新的代码库。 
在你自己的[github](https://github.com "Github")主页上创建一个新的代码库，取名为：USERNAME.github.com，其中USERNAME为你自己的用户名。
###3.2 安装Jekyll-Bootstrap
找一个合适的位置，使用Git bash输入以下命令：

    git clone https://github.com/plusjade/jekyll-bootstrap.git USERNAME.github.com
    cd USERNAME.github.com
    git remote set-url origin git@github.com:USERNAME/USERNAME.github.com.git
    git push origin master
###3.3 结果
等待几分钟之后，在浏览器中输入http://USERNAME.github.com，你就会看到Jekyll-Bootstrap的样例博客了,gorgeous!如果你希望在本地来调试你的博客的话，只要在在Git bash中运行：

    $ cd USERNAME.github.com 
    $ jekyll serve
打开[http://localhost:4000/](http://localhost:4000/)上就可以看到效果了，这样你就可以在本地调试好之后push到github上。  
##4 修改  
###4.1 基本结构
接下来我们要做的就是在这个基础上把它修改成我们自己的博客了。我们介绍一个最基础的Jekyll博客的目录结构：
    
    ├── _config.yml
    ├── _drafts
    ├── _includes
    ├── _layouts
    ├── _posts
    ├── _site
    └── index.html
这些目录的介绍如下：  
<table>
    <tr>
        <td>目录/文件</td><td>描述</td>
    </tr>
    <tr>
        <td>_config.yml</td><td>存储配置数据。很多全局的配置或者指令写在这里。</td>
    </tr>
    <tr>
        <td>_drafts</td><td>存放为发表的文章。这些是没有日期的文件。</td>
    </tr>
    <tr>
        <td>_includes</td><td>存放一些组件。可以通过{% include file.ext %} 来引用。  </td>
    </tr>
    <tr>
        <td>_layouts</td><td>布局。</td>
    </tr>
    <tr>
        <td>_posts</td><td>存放写文章，格式化为：YEAR-MONTH-DAY-title.md。  </td>
    </tr>
    <tr>
        <td>_site</td><td>最终生成的博客文件就在这里。</td>
    </tr>
    <tr>
        <td>index.html</td><td>博客的主页。</td>
    </tr>
</table>
###4.2 开始修改
首先我们编辑_config.yml文件，将页面的一些基本参数改掉：

    title : My Blog =)
    
    author :
      name : Name Lastname
      email : blah@email.test
      github : username
      twitter : username

比如页眉显示的文字，以及页脚显示的作者，邮件等等。在_posts文件夹下有个core-samples文件夹存放的简单样例，我们可以把它删掉：

    rm -rf _posts/core-samples
这时候我们运行目录jekyll build一下，发现主页并没有被改变还是显示的关于Jekyll-Bootstrap的，于是我们打开index.md，修改里面的内容，将与Bootstrap相关的东西全部删除，只留下posts list用来显示我们的博客列表。再打开README.md修改下readme，这样我们的博客框架就修改好了。
接下来做的就是将代码push到github就可以了。

    git add --all
    git commit -m "change the content"
    git push origin master
###4.3 创建文章
我们可以通过目录来创建文章：

    rake post title="Hello World"
这个命令会在_posts文件夹下创建一个year-month-day-hello-world.md文件,打开之后我们会发现yaml头已经自动生成好了，我们只需要选择自己喜欢的编辑器就可以开始创作了。
###4.4 创建页面
除了可以创建文章，我们还可以创建一些其他的页面。

    rake page name="about.md"
    rake page name="pages/about.md"
##5 问题
在过程中，也遇到了一些问题，整理如下：
###5.1 关于windows下git bash的中文显示。
解决方法：  
1、修改C:\Program Files(x86)\Git\etc\git-completion.bash,添加  

    	alias ls='ls --show-control-chars --color=auto'  
说明：使得在 Git Bash 中输入 ls 命令，可以正常显示中文文件名。  
2、修改C:\Program Files(x86)\Git\etc\inputrc，添加

    set output-meta on
    set convert-meta off
说明：使得在 Git Bash 中可以正常输入中文，比如中文的 commit log。
3、修改C:\Program Files(x86)\Git\etc\profile，添加

    export LESSCHARSET=utf-8
说明：$ git log 命令不像其它 vcs 一样，n 条 log 从头滚到底，它会恰当地停在第一页，按 space 键再往后翻页。这是通过将 log 送给 less 处理实现的。以上即是设置 less 的字符编码，使得 $ git log 可以正常显示中文。其实，它的值不一定要设置为 utf-8，比如 latin1 也可以……。还有个办法是 $ git –no-pager log，在选项里禁止分页，则无需设置上面的选项。  
4、修改C:\Program Files\Git\etc\gitconfig，添加

    [gui]
    	encoding = utf-8
说明：我们的代码库是统一用的 utf-8，这样设置可以在 git gui 中正常显示代码中的中文。

    [i18n]
 	    commitencoding = GB2312
说明：如果没有这一条，虽然我们在本地用 $ git log 看自己的中文修订没问题，但是，一、我们的 log 推到服务器后会变成乱码；二、别人在 Linux 下推的中文 log 我们 pull 过来之后看起来也是乱码。这是因为，我们的 commit log 会被先存放在项目的 .git/COMMIT_EDITMSG 文件中；在中文 Windows 里，新建文件用的是 GB2312 的编码；但是 Git 不知道，当成默认的 utf-8 的送出去了，所以就乱码了。有了这条之后，Git 会先将其转换成 utf-8，再发出去，于是就没问题了。
###5.2 修改jekyll_bootstrap主页的yaml头时出现中文字符不能build成功
这是估计因为字符集不兼容的问题，解决方法：  
修改C:\Program Files (x86)\Git\etc路径下的git-completion.bash文件： 添加  

    export LC_ALL=zh_CN.UTF-8
    export LANG=zh_CN.UTF-8
