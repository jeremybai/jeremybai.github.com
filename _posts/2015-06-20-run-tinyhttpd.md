---
layout: post
title: "轻量级服务器tinyhttpd源码分析－本地运行"
description: ""
categories: 
- tinyhttpd
- c
tags: []
---

　　从[sourceforge](http://sourceforge.net/projects/tinyhttpd/)上下载源码到本地，我在看源码之前喜欢先将程序运行起来看下程序运行的效果，这样对于程序的功能先有一个感性的认识。  
　　下载下来的源码是在Solaris 2.6上编译运行的，在httpd.c中写道：在如果你想要在Linux上运行的话，需要进行一些修改：    
　　
	1) 注释掉 #include <pthread.h>   
	2) 注释掉newthread变量的定义  
	3) 注释掉pthread_create()函数的调用  
	4) 将accept_request()的调用取消注释  
	5) 在Makefile中移除-lsocket  
　　修改完之后Server变成了单线程的程序，只能够一个客户端进行连接，所以决定不按照上面的修改，本机的运行环境：OS X。首先将程序make下看看有什么错误。  
{% highlight c %}
httpd.c:437:52: warning: passing 'int *' to parameter of type 'socklen_t *' (aka 'unsigned int *') converts between pointers to integer types with different
      sign [-Wpointer-sign]
  if (getsockname(httpd, (struct sockaddr *)&name, &namelen) == -1)
                                                   ^~~~~~~~
/usr/include/sys/socket.h:566:74: note: passing argument to parameter here
int     getsockname(int, struct sockaddr * __restrict, socklen_t * __restrict)
                                                                             ^
httpd.c:491:24: warning: passing 'int *' to parameter of type 'socklen_t *' (aka 'unsigned int *') converts between pointers to integer types with different
      sign [-Wpointer-sign]
                       &client_name_len);
                       ^~~~~~~~~~~~~~~~
/usr/include/sys/socket.h:560:69: note: passing argument to parameter here
int     accept(int, struct sockaddr * __restrict, socklen_t * __restrict)
                                                                        ^
httpd.c:495:40: warning: incompatible pointer types passing 'void (int)' to parameter of type 'void *(*)(void *)' [-Wincompatible-pointer-types]
 if (pthread_create(&newthread , NULL, accept_request, client_sock) != 0)
                                       ^~~~~~~~~~~~~~
/usr/include/pthread.h:314:11: note: passing argument to parameter here
                void *(*)(void *), void * __restrict);
                        ^
httpd.c:495:56: warning: incompatible integer to pointer conversion passing 'int' to parameter of type 'void *' [-Wint-conversion]
 if (pthread_create(&newthread , NULL, accept_request, client_sock) != 0)
                                                       ^~~~~~~~~~~
/usr/include/pthread.h:314:39: note: passing argument to parameter here
                void *(*)(void *), void * __restrict);
                                                    ^
4 warnings generated.
{% endhighlight %} 
　　有四个警告，让我们一一解决掉，首先是：  
{% highlight c %}
httpd.c:437:52: warning: passing 'int *' to parameter of type 'socklen_t *' (aka 'unsigned int *') converts between pointers to integer types with different
{% endhighlight %} 　　
　　这个警告的意思就是getsockname函数的第三个参数应该是socklen_t *类型，但是实际传入的参数类型是int *类型，只需要修改下namelen定义的类型修改为socklen_t类型。  
{% highlight c %}
httpd.c:491:24: warning: passing 'int *' to parameter of type 'socklen_t *' (aka 'unsigned int *') converts between pointers to integer types with different
{% endhighlight %} 
　　这个错误与上面错误类似，也是类型不匹配，将client_name_len定义为socklen_t类型就好了。  
{% highlight c %}
httpd.c:495:40: warning: incompatible pointer types passing 'void (int)' to parameter of type 'void *(*)(void *)' [-Wincompatible-pointer-types]
{% endhighlight %} 
　　这个错误也是参数类型不匹配，pthread_create函数的第三个参数是函数指针类型，这个函数指针应该是void *(*)(void *)类型，但是定义的accept_request声明为；  
　　
	void accept_request(int);  
　　所以需要对accept_request进行一些修改，首先将accept_request声明为：  
　　
	void *accept_request(void *);  
　　因为参数类型和函数返回值都改变了，所以在函数内部需要做些改动，在client使用之前将ptr_client转为client，还有就是需要在最后增加返回语句return NULL;，在中间需要返回的时候也需要返回NULL，修改完的accept_request代码如下：  
{% highlight c %}
void *accept_request(void *pclient)
{
    char buf[1024];
    int numchars;
    char method[255];
    char url[255];
    char path[512];
    size_t i, j;
    struct stat st;
    int cgi = 0;      /* becomes true if server decides this is a CGI program */
    char *query_string = NULL;
    size_t client = *(size_t *)pclient;

    numchars = get_line(client, buf, sizeof(buf));
    i = 0; j = 0;
    while (!ISspace(buf[j]) && (i < sizeof(method) - 1))
    {
        method[i] = buf[j];
        i++; j++;
    }
    method[i] = '\0';

    if (strcasecmp(method, "GET") && strcasecmp(method, "POST"))
    {
        unimplemented(client);
        return NULL;
    }

    if (strcasecmp(method, "POST") == 0)
        cgi = 1;

    i = 0;
    while (ISspace(buf[j]) && (j < sizeof(buf)))
        j++;
    while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < sizeof(buf)))
    {
        url[i] = buf[j];
        i++; j++;
    }
    url[i] = '\0';

    if (strcasecmp(method, "GET") == 0)
    {
        query_string = url;
        while ((*query_string != '?') && (*query_string != '\0'))
            query_string++;
        if (*query_string == '?')
        {
            cgi = 1;
            *query_string = '\0';
            query_string++;
        }
    }

    sprintf(path, "htdocs%s", url);
    if (path[strlen(path) - 1] == '/')
        strcat(path, "index.html");
    if (stat(path, &st) == -1) {
    while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
        numchars = get_line(client, buf, sizeof(buf));
        not_found(client);
    }
    else
    {
        if ((st.st_mode & S_IFMT) == S_IFDIR)
            strcat(path, "/index.html");
        if ((st.st_mode & S_IXUSR) ||
            (st.st_mode & S_IXGRP) ||
            (st.st_mode & S_IXOTH)    )
            cgi = 1;
        if (!cgi)
            serve_file(client, path);
        else
            execute_cgi(client, path, method, query_string);
    }

    close(client);
    return NULL;
}
{% endhighlight %} 
　　再将-lsocket从Makefile中删去即可。  
　　最后，还有一处错误需要修改：  
　　
	execl(path, path, NULL);  
　　改为  
　　
	execl(path, query_string, NULL);  
　　再次运行make，没有警告了，运行可执行文件httpd：./ httpd，显示程序在哪个端口监听，打开浏览器，输入127.0.0.1:<端口号>，页面如图所示：  
![](http://7fv9jl.com1.z0.glb.clouddn.com/2015-06-20-run-tinyhttpd-1.png)  
　　在框中输入red或者blue，点击按钮跳转，跳转之后的页面显示为你填写的颜色：  
![](http://7fv9jl.com1.z0.glb.clouddn.com/2015-06-20-run-tinyhttpd-2.png)  
　　经过上述的修改tinyhttpd便能在本地运行起来了，接下来对它的源码进行简要的分析。  