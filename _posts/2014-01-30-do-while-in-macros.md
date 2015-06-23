---
layout: post
title: "宏里面的do {...} while (0)"
description: ""
categories: 
- C
- 翻译
tags: []
---
　　原文链接：http://www.pixelstech.net/article/1390482950-do-%7B-%7D-while-%280%29-in-macros  

----------

　　如果你是一个C程序员，想必你一定会对宏很熟悉。它们威力无穷，如果使用得当可以很大程度简化我们的工作。但是，如果你在定义宏的时候不是那么的小心，那么它们就会咬你甚至把你逼疯。在许多的C代码中，你或许看到过这样一个看起来不是那么直接，让你觉着奇怪的宏定义：
{% highlight c++ %}
#define __set_task_state(tsk, state_value)      \
    do { (tsk)->state = (state_value); } while (0)
{% endhighlight %}

　　如果你看过Linux内核或者其他比较出名的C库的源码，那你一定对这种写法不陌生。那么这种写法到底有什么用呢？ 来自谷歌网页搜索部门的[Robert Love](http://www.quora.com/Robert-Love-1)（曾做过Linux内核开发）给了我们答案。

　　do{...}while(0)是唯一可以在任何情况下（尤其实在没有花括号的if语句中）都让你的宏作用一样的结构，即使你的宏后面带上了分号。例如：
{% highlight c++ %}
#define foo(x)  bar(x); baz(x)
{% endhighlight %}
　　在定义了这个之后，你接下来可能会这么使用它：
{% highlight c++ %}
foo(wolf);
{% endhighlight %}
　　那么这条宏定义便会被展开为：
{% highlight c++ %}
bar(wolf); baz(wolf);
{% endhighlight %}
　　在这种情况下，这个结果是我们所希望的。然而，如果我们是这样使用的：
{% highlight c++ %}
if (!feral)
	foo(wolf);
{% endhighlight %}
　　上面那个表达式便会被展开为：
{% highlight c++ %}
if (!feral)
	bar(wolf);
baz(wolf);
{% endhighlight %}
　　多语句宏并不是在什么时候都有用，在这种情况下，只有do/while(0)才能达到你的目的！
　　如果我们用do/while(0)重新定义之前那个宏：
{% highlight c++ %}
#define foo(x)  do { bar(x); baz(x); } while (0)
{% endhighlight %}
　　在功能上看，我们发现是与之前那个宏定义等价的。do保证了花括号内代码的执行。 while(0) 保证了这段代码只被执行一次。和没有循环一样。我们再将这个宏应用于上面的if语句，就会变成这样：
{% highlight c++ %}
if (!feral)
    do { bar(wolf); baz(wolf); } while (0);
{% endhighlight %}
　　从语义上讲，是和下面的代码等价的：
{% highlight c++ %}
if (!feral) {
    bar(wolf);
    baz(wolf);
}
{% endhighlight %}
　　或许你又会辩解，直接用花括号将代码块包起来就可以了，为什么又要在外面套上一层do/while(0)呢？举个例子你就明白了，我们将之前的宏定义重新定义为：
{% highlight c++ %}
#define foo(x)  { bar(x); baz(x); }
{% endhighlight %}
　　在之前的if语句的情况下展开之后的结果是我们所希望的，但是如果遇到这样使用情况：
{% highlight c++ %}
if (!feral)
    foo(wolf);
else
    bin(wolf);
{% endhighlight %}
　　只用花括号包起来的代码就会被展开为：
{% highlight c++ %}
if (!feral) {
    bar(wolf);
    baz(wolf);
};
else
    bin(wolf);
{% endhighlight %}
　　这在编译的时候会报错的，因为有语义错误。

**　　总而言之，在Linux或者其他的代码中将宏用do/while(0)包起来就是为了无论在代码中分号和花括号如何使用都能确保宏的效果始终一致。**
