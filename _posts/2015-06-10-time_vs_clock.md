---
layout: post
title: "Python中time.clock()和time.time()的区别"
description: ""
categories: 
- python
tags: []
---

## CPU time和wall time的区别
　　有时候需要统计程序的运行时间，这是我们一般会做一个艰难的选择：是使用time.clock()还是time.time()？网上搜了下，答案一大堆，却没有看出什么头绪，查了一些材料，首先需要明确几个概念：`CPU time`和`wall time`。  
　　`CPU time`是当CPU完全被某个进程所使用时所花费的时间，因为CPU并不是被某个进程单独占用的，在你的进程执行的这段时间中，你的进程可能只占用了其中若干的时间片（由操作系统决定），CPU时间只是处理你的进程占用的那些时间片的相加，对于这段时间中由其他进程占用的时间片是不纳入你的进程的CPU时间的。  
　　`wall time`从名字上来看就是墙上时钟的意思，可以理解为进程从开始到结束的时间，包括其他进程占用的时间。  
　　举个[例子](https://service.futurequest.net/index.php?/Knowledgebase/Article/View/407)来说，当使用gzip压缩一个20MB的Apache日志文件时我们可以看下花费时间：  

	$ time gzip access.log
	real 0m2.125s
	user 0m1.920s
	sys 0m0.170s  
　　这个进程总共花费了2.125秒的wall time（"real"）、1.920s的CPU time（"user"）、在内核态占用了0.170s（"sys"）,剩下的0.035s是被其他进程所使用的时间片。可以看出gzip的过程占用了较多的CPU time，有些其他的任务则会占用较多的wall time，比如说`find`命令：  

	$ time find /big/dom -ls
	real 0m37.707s
	user 0m1.210s
	sys 0m1.260s
　　这个进程的wall time和CPU time相差了35.237s，这些时间都被其他进程所占用，原因是因为这个进程需要在屏幕上打印出超过50000个文件，打印的过程中CPU是空闲的，因此调度系统便将时间片分配给了其他的进程。如果使用`$ time find /big/dom -ls > /dev/null`这个命令来进行查找，则会发现wall time和CPU time相差无几，因为省去了输出文件名到屏幕的时间。  

	$ time find /big/dom -ls > /dev/null
	real 0m1.164s
	user 0m0.900s
	sys 0m0.260s     
## time()和clock()函数说明
　　了解了上面的相关知识接下来回到正题，Python的[time模块文档](https://docs.python.org/2/library/time.html#time.clock)，clock()的函数说明如下所示：  
time.time()：  

	Return the time in seconds since the epoch as a floating point number. Note that even though the time is always returned as a floating point number, not all systems provide time with a better precision than 1 second. While this function normally returns non-decreasing values, it can return a lower value than a previous call if the system clock has been set back between the two calls.
 
　　time()函数返回的是从Epoch（在Linux上是1970年1月1日00:00:00 UTC，在Windows上可能是1601年1月1日00:00:00 UTC，可以通过`time.gmtime(0)`函数获取当前系统的Epoch时间）开始所经过的秒数。尽管这个函数返回的时间是浮点数，但是并不意味着所有的系统都能够提供小于1秒的精度。虽然这个额函数返回的是递增的值，如果系统时钟被设置之后该函数的返回值可能比之前一次调用小。  

time.clock()：  

	On Unix, return the current processor time as a floating point number expressed in seconds. The precision, and in fact the very definition of the meaning of “processor time”, depends on that of the C function of the same name, but in any case, this is the function to use for benchmarking Python or timing algorithms.
	
	On Windows, this function returns wall-clock seconds elapsed since the first call to this function, as a floating point number, based on the Win32 function QueryPerformanceCounter(). The resolution is typically better than one microsecond.
　　clock函数在`unix系统`中，以秒的形式返回当前的进程时间（浮点数类型），其精度取决于相同名字的C函数的精度，是用来对Python或时间算法进行基准测试的函数。在`Windows系统`中，该函数返回自从第一次调用clock()之后经过的时间，该函数基于Win32的QueryPerformanceCounter()函数，分辨率通常小于一微秒。  
##结论
　　time()在`unix系统`和`Windows系统`中返回的值一样，都是从Epoch开始所经过的秒数，所以在使用time()函数时不需要考虑系统的差异性，但是使用clock()函数时需要注意，在不同的操作系统上的返回值代表不同的含义：    
　　在`unix系统`中每次调用clock()函数返回的值是一样的，都返回当前进程的`CPU时间`（不包括其他进程占用的时间或者等待的时间）；     
　　在`Windows系统`中，clock()函数第一次调用时返回的是当前进程的`CPU时间`，之后的调用返回自从第一次调用clock()之后经过的时间，所以当我们需要得到程序的运行时间时，调用两次clock()，将得到的结果相减，代码如下：      
{% highlight python %}
#!/usr/bin/env python
# -*- coding:utf-8 -*-
import time

def procedure():
    time.sleep(2.5)

t0 = time.clock()
procedure()
t1 = time.clock()
print "time.clock(): ", t1 - t0

t0 = time.time()
procedure()
t1 = time.time()
print "time.time()", t1 - t0
{% endhighlight %} 
　　在`Windows系统`中上述的方法都可以获得procedure()运行的时间，如果你想获取程序的CPU时间，第一次调用clock()的返回值就是；如果在`unix系统`中，两次调用clock()的结果都是一样的，所以第一种方法是没有效果的，如果你想获取程序的CPU时间，无论哪一次调用clock()的返回值都是CPU时间。   

###参考文献：
[1] [Python time模块文档](https://docs.python.org/2/library/time.html#time.clock)  
[2] [Difference between CPU time and wall time](https://service.futurequest.net/index.php?/Knowledgebase/Article/View/407)  
[3] [Measure Time in Python – time.time() vs time.clock()](http://pythoncentral.io/measure-time-in-python-time-time-vs-time-clock/)
