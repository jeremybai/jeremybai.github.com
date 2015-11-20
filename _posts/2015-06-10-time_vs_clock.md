---
layout: post
title: "Python中time.clock()和time.time()的区别"
description: ""
categories: 
- python
tags: []
---

　　有时候需要统计程序的运行时间，这是我们一般会做一个艰难的选择：是使用time.clock()还是time.time()？网上搜了下，答案一大堆，却没有看出什么头绪，查了Python的time模块文档，clock()的函数说明如下所示：  
time.clock()：  

	On Unix, return the current processor time as a floating point number expressed in seconds. The precision, and in fact the very definition of the meaning of “processor time”, depends on that of the C function of the same name, but in any case, this is the function to use for benchmarking Python or timing algorithms.
	
	On Windows, this function returns wall-clock seconds elapsed since the first call to this function, as a floating point number, based on the Win32 function QueryPerformanceCounter(). The resolution is typically better than one microsecond.

time.time()：  

	Return the time in seconds since the epoch as a floating point number. Note that even though the time is always returned as a floating point number, not all systems provide time with a better precision than 1 second. While this function normally returns non-decreasing values, it can return a lower value than a previous call if the system clock has been set back between the two calls.
　　
　　clock函数在unix系统中，以秒的形式返回当前的进程时间（浮点数类型），其精度取决于相同名字的C函数的精度，是用来对Python或时间算法进行基准测试的函数。在Windows中，该函数返回自从上次调用clock()之后经过的时间，该函数基于Win32的QueryPerformanceCounter()函数，分辨率通常小于一微秒。  
　　time()函数返回的是从Epoch（1970年1月1日00:00:00 UTC）开始所经过的秒数。尽管这个函数返回的时间是浮点数，但是并不意味着所有的系统都能够提供小于1秒的精度。虽然这个额函数返回的是递增的值，如果系统时钟被设置之后该函数的返回值可能比之前一次调用小。  
　　所以在计算程序执行的时间时，使用clock()函数时不需要减去第一次调用clock()返回的时间因为第二次调用时返回的就是从上次调用clock()之后经过的时间。如果使用time()函数时需要将两次调用的返回值相减得到运行时间，代码如下：    
{% highlight python %}
#!/usr/bin/env python
# -*- coding:utf-8 -*-
import time

def procedure():
    time.sleep(2.5)

t0 = time.clock()
procedure()
t1 = time.clock()
print "time.clock(): ", t1

t0 = time.time()
procedure()
t1 = time.time()
print "time.time()", t1 - t0
{% endhighlight %} 