---
layout: post
title: "python爬虫获取豆瓣电影——多线程问题"
description: ""
categories: 
- python
- web
tags: [豆瓣,API]
---

　　GIL全称Global Interpreter Lock（全局解释器锁），它是在实现Python解析器(CPython)时所引入的一个概念。但是它并不是Python的特性。Python是一种语言，它有自己的语法等规范，根据其实现的不同有Cpython, Jython等等，CPython是使用C语言编写Python以及相关的解释器，Jython是用Java来编写的。Python完全可以不依赖于GIL，比如说JPython解析器就没有GIL。因为大部分环境下Python执行环境默认为CPython，许多人便把GIL归结为Python的特性。  
　　在[Python的wiki](https://wiki.python.org/moin/GlobalInterpreterLock)中是这样定义GIL的：  

	In CPython, the global interpreter lock, or GIL, is a mutex that prevents multiple native threads from executing Python bytecodes at once. This lock is necessary mainly because CPython's memory management is not thread-safe. (However, since the GIL exists, other features have grown to depend on the guarantees that it enforces.)
　　翻译过来也就是说，GIL是一个互斥锁用来防止多个线程并发执行Python字节码，这把锁存在原因是CPython的内存管理不是线程安全的（因为GIL的存在其他一些功能已经习惯并依赖于它了）。如果你想要摆脱GIL的束缚，可以使用多进程（multiprocess库）来实现你的代码。    
　　由于GIL的存在导致在任何时刻只有一个线程在执行，但是这并不意味着多线程编程在Python中一无是处，当爬虫的多个线程的其中一个在爬取网站时阻塞在等待服务器的响应时，其他线程的任务便可以利用阻塞任务等待的时间进行其他操作以提高效率。操作数据库时也存在同样的问题，有可能你需要等待数据库返回的结果，在等待期间你可以利用这段时间做其它的一些操作。下面我给出了一个多线程的最基本的框架，当我们实现了单个线程的程序之后，直接移植过来即可，同时还需要考虑到多线程可能会出现的问题，比如你的操作是否是原子操作，这些在后面详细设计中都会详细的介绍。这里给出一个最简单的多线程实现的框架，首先构造一个队列，将参数入队，新建若干线程，线程从队列中获取参数然后执行响应的操作，最后等待所有任务完成之后退出：   
{% highlight python %}
#!/usr/bin/env python
# -*- coding:utf-8 -*-
import threading
import Queue
import time
import sys

reload(sys)
sys.setdefaultencoding('utf8')

q = Queue.Queue()   #构造一个不限制大小的的队列
thread_num = 10     #设置线程的个数

#具体的处理函数，负责处理单个任务
def do_somthing_using(arguments):
    print arguments

def worker():
    global q
    while not q.empty():
        arguments = q.get()
        do_somthing_using(arguments)
        time.sleep(1)
        q.task_done()

def main():
    global q
    threads = []
    for index in xrange(10):
        q.put(index)
    for i in xrange(thread_num):
        thread = threading.Thread(target=worker)
        # 线程开始处理任务
        thread.start()
        threads.append(thread)
    # 等待线程池中的线程全部结束
    for thread in threads:
        thread.join()
    # 等到队列为空，再执行下面的操作
    q.join()

if __name__ == '__main__':
    start = time.clock()
    main()
    end = time.clock()
    print "共花费: %f s" % (end - start)
{% endhighlight %} 
　　 

----------
　　除了使用threading+Queue的方法实现多线程，还有种更简单的方法：map，使用map可以通过几行代码就实现并行化。感兴趣的同学可以参考[《一行 Python 实现并行化 -- 日常多线程操作的新思路》](http://segmentfault.com/a/1190000000414339)这篇文章。  
　　map使用起来相当简单，下面的例子介绍下map的简单使用，首先通过ThreadPool创建线程池，它决定了你的线程池中线程的数目，默认值为当前机器CPU的核数，通过map函数就可以将参数序列中的参数分配到每个线程对应的执行函数中去，map函数的返回值是一个列表，记录了每个线程执行完任务函数返回的结果。    
{% highlight python %}
#!/usr/bin/env python
# -*- coding:utf-8 -*-
import time
import sys
from multiprocessing.dummy import Pool as ThreadPool

reload(sys)
sys.setdefaultencoding('utf8')

#具体的处理函数，负责处理单个任务
def do_somthing_using(arguments):
    print arguments
    time.sleep(1)
    return arguments

def run():
    # 创建线程池
    pool = ThreadPool(5)
    # 将第list中的每个元素作为参数传递到do_somthing_using方法中
    # 并将所有结果保存到results这一列表中。
    list = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    results = pool.map(do_somthing_using, list)
    # 调用join之前先调用close函数，否则会出错
    # 执行完close后不会有新的进程加入到pool,join函数等待所有线程结束
    pool.close()
    pool.join()

if __name__ == '__main__':
    start = time.clock()
    run()
    end = time.clock()
    print "共花费: %f s" % (end - start)
{% endhighlight %} 