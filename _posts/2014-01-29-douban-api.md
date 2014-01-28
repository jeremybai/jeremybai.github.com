---
layout: post
title: "如何调用豆瓣API"
description: ""
categories: 
- Javascript
- JQuery
- web
tags: [豆瓣,API]
---
{% include JB/setup %}

　　最近在自己的网页中希望可以输入书籍的ISBN号就可以自动补全书籍的所有信息，人工输入工作量太大，就选择了调用豆瓣的API获得数据填写到表单中。做了两天才做好，网上有用的资料比较少，于是晒出自己这两天做的一些东西，希望可以帮助到需要的人，注意：APIV1.0和2.0是有区别的。在下面有提到，需要稍微留意。  
　　下面的代码实现的功能都是将ISBN号为9787543632608（片山恭一的[《满月之夜白鲸现》](http://book.douban.com/subject/1220562/)）的图书信息返回出来，只是使用的方法不一样而已，运行的结果如下图,贴html代码似乎有点问题（已解决，参见文章末解决办法），还没找到解决办法，只能先转成图片上传，代码点击[这里](https://github.com/jeremybai/jeremybai.github.com/blob/master/images/2014-01-29/doubanapi.html)：  

![]({{site.img_url}}/2014-01-29/doubanapi.jpg) 

## 1. JS调用 ##

　　通过豆瓣js中的函数来获得图书的信息。JS文件采用的是豆瓣的apiV2.0，不过返回出来结果和旧版的apiV1.0一样，返回的信息不是JSON格式，需要调用parseSubject函数来转为JSON对象。代码如下：

    <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.0 Transitional//EN">
    <HTML xmlns="http://www.w3.org/1999/xhtml">
    <HEAD>
    <TITLE></TITLE>
    <meta content="text/html; charset=utf-8" http-equiv="Content-Type">
    <script src="http://ajax.googleapis.com/ajax/libs/jquery/1.5.2/jquery.min.js" type="text/javascript"></script>
    <script type="text/javascript" src="http://www.douban.com/js/api.js?v=2"></script>
    <script type="text/javascript" src="http://www.douban.com/js/api-parser.js?v=1"></script>
    </HEAD>
    <BODY>
    </BODY>
    <script>
    DOUBAN.apikey = '00fa6c0654689a0202ef4412fd39ce06'
    DOUBAN.getISBNBook({
    isbn:'9787543632608',
    callback:function(book1){
       var subj = DOUBAN.parseSubject(book1)
       var tl = subj.title ? subj.title : "";
       var author = subj.author ? subj.author : "";
       var tmp = "<img src="+subj.link.image+" style='margin:10px;float:left'>";
       tmp += "<div>Title : <a href="+subj.link.alternate+" target='_blank'>"+tl+"</a></div>";
       if (subj.attribute.author) tmp += "<div>Authors : "+(subj.attribute.author.join(' / '))+"</div>";
       if (subj.attribute.isbn13) tmp += "<div>ISBN : "+(subj.attribute.isbn13.join(' / '))+"</div>";
       if (subj.attribute.price) tmp += "<div>Price : "+(subj.attribute.price.join(' <br/>   '))+"</div>";
       if (subj.attribute.pages) tmp += "<div>Pages : "+(subj.attribute.pages.join(' / '))+"</div>";
       if (subj.attribute.publisher) tmp += "<div>Publisher : "+(subj.attribute.publisher.join(' <br/>   '))+"</div>";
       if (subj.attribute.pubdate) tmp += "<div>Pubdate : "+(subj.attribute.pubdate.join(' / '))+"</div>";
       if (subj.rating.average)
    tmp +="<div>Rating: "+subj.rating.average+" / "+subj.rating.numRaters+decodeURI("%E4%BA%BA")+ "</div>"
       tmp += "<p>"+(subj.summary ? subj.summary : "")+"</p>";
       document.body.innerHTML = tmp;
    }
    })
    </script>
    </HTML>   

## 2. JQUERY调用 ##
　　替换上面代码的script标签里面的内容。getJSON函数的第一个参数为url，第二个为回调函数，即获取url的JSON数据之后执行的函数，这个url是老版本的apiV1.0，所以同上面一样，还是要调用parseSubject函数。

    <script type="text/javascript">
    $.getJSON("http://api.douban.com/book/subject/1220562?alt=xd&callback=?", function(book) {
       var subj = DOUBAN.parseSubject(book);
       var tl = subj.title ? subj.title : "";
       var author = subj.author ? subj.author : "";
       var tmp = "<img src="+subj.link.image+" style='margin:10px;float:left'>";
       tmp += "<div>Title : <a href="+subj.link.alternate+" target='_blank'>"+tl+"</a></div>";
       if (subj.attribute.author) tmp += "<div>Authors : "+(subj.attribute.author.join(' / '))+"</div>";
       if (subj.attribute.isbn13) tmp += "<div>ISBN : "+(subj.attribute.isbn13.join(' / '))+"</div>";
       if (subj.attribute.price) tmp += "<div>Price : "+(subj.attribute.price.join(' <br/>   '))+"</div>";
       if (subj.attribute.pages) tmp += "<div>Pages : "+(subj.attribute.pages.join(' / '))+"</div>";
       if (subj.attribute.publisher) tmp += "<div>Publisher : "+(subj.attribute.publisher.join(' <br/>   '))+"</div>";
       if (subj.attribute.pubdate) tmp += "<div>Pubdate : "+(subj.attribute.pubdate.join(' / '))+"</div>";
       if (subj.rating.average)
    tmp +="<div>Rating: "+subj.rating.average+" / "+subj.rating.numRaters+decodeURI("%E4%BA%BA")+ "</div>"
       tmp += "<p>"+(subj.summary ? subj.summary : "")+"</p>";
       document.body.innerHTML = tmp;
    });
    </script>

　　如果采用的是APIV2.0版，有些不同，首先getJSON函数url参数是新版的url，返回出来的数据格式就是JSON格式，不需要转换直接使用即可。有个问题，除了book.author可以加上.join(' / ')加上分隔符，其他参数加上这个函数就会出错，不知道什么原因。
    <script type="text/javascript">
    $.getJSON("https://api.douban.com/v2/book/1220562?alt=xd&callback=?", function(book) {
       var title = book.title ? book.title : "";
       var tmp = "<img src="+book.image+" style='margin:10px;float:left'>";
       tmp += "<div>Tiktle : <a href="+book.alt+" target='_blank'>"+title+"</a></div>";
       if (book.author) tmp += "<div>Authors : "+book.author+"</div>";
       if (book.pubdate) tmp += "<div>Pubdate : "+book.pubdate+"</div>";
       if (book.isbn13) tmp += "<div>ISBN : "+book.isbn13+"</div>";
       if (book.price) tmp += "<div>Price : "+book.price+"</div>";
       if (book.pages) tmp += "<div>Pages : "+book.pages+"</div>";
       if (book.publisher) tmp += "<div>Publisher : "+book.publisher+"</div>";
       if (book.rating.average) tmp +="<div>Rating: "+book.rating.average+" / "+book.rating.numRaters+decodeURI("%E4%BA%BA")+ "</div>"
       tmp += "<p>"+(book.summary ? book.summary : "")+"</p>";
       document.body.innerHTML = tmp;
    });
    </script>  
  

##附：JSON格式 ##
　　关于JSON格式，这个是在网上看到阮一峰老师写的[博客](http://www.ruanyifeng.com/blog/2009/05/data_types_and_json.html)的，写的很好，一下子就理解了。

>　　从结构上看，所有的数据（data）最终都可以分解成三种类型：  
>　　第一种类型是**标量** （scalar），也就是一个单独的字符串（string）或数字（numbers），比如"北京"这个单独的词。  
>　　第二种类型是**序列** （sequence），也就是若干个相关的数据按照一定顺序并列在一起，又叫做数组（array）或列表（List），比如"北京，上海"。  
>　　第三种类型是**映射** （mapping），也就是一个名/值对（Name/value），即数据有一个名称，还有一个与之相对应的值，这又称作散列（hash）或字典（dictionary），比如"首都：北京"。  
>　　21世纪初，Douglas Crockford寻找一种简便的数据交换格式，能够在服务器之间交换数据。当时通用的数据交换语言是XML，但是Douglas Crockford觉得XML的生成和解析都太麻烦，所以他提出了一种简化格式，也就是Json。  
　　Json的规格非常简单，只用一个页面几百个字就能说清楚，而且Douglas Crockford声称这个规格永远不必升级，因为该规定的都规定了。  
>　　1） 并列的数据之间用逗号（", "）分隔。  
>　　2） 映射用冒号（": "）表示。  
>　　3） 并列数据的集合（数组）用方括号("[]")表示。  
>　　4） 映射的集合（对象）用大括号（"{}"）表示。  
>　　上面四条规则，就是Json格式的所有内容。  

## jekyll中的不能嵌套html代码 ##
之前不能在markdown文件中嵌入html代码，在build的时候总会出问题，貌似是转义字符的问题，现在可以了，解决方法是使用RDiscount模板解释器替代Maruku。这个解释器到底是干嘛用的呢？和其它轻量标记语言一样，Markdown并不能也不旨在替代HTML；因为所有的网页最终都要交给浏览器来解析的，而浏览器只认识HTML，因此，单独使用Markdown编写的文本并不能为浏览器所认识；所以，在浏览器和Markdown之间还要有一个东西，那就是翻译器。它将已完成的Markdown格式的文本转换成HTML文本，然后才能交由浏览器来解析和排版。Markdown官方发布的翻译器是一个Perl脚本，在命令行执行如下命令即可转换Markdown文本到HTML文本：

    perl Markdown.pl --html4tags sample.txt > sample.html  

Maruku是纯ruby的Markdown模版解释器；RDiscount则是c写的模版解释器，[两者的效率显然不同](http://stackoverflow.com/questions/373002/better-ruby-markdown-interpreter)。RDiscount的安装很简单：  

    gem install RDiscount

再修改下_config.yml文件，加上：

    markdown: rdiscount
再运行`jekyll build`和`jekyll server`就不会有错误了。