---
layout: post
title: "Python中包的导入"
description: ""
categories: 
- python
tags: []
---

　　当你从Python解释器退出后再重新进入, 之前所写的代码，包括变量、函数都已经不存在了，如果你想周期性的执行这些功能但是并不想每次都将这些代码重新输入一遍，你可以将这些代码保存成文件在本地进行存储（也就是脚本），当你写的程序规模越来越大，维护起来越发吃力，你或许会想把它分割为不同功能的文件，再或者你需要在其他项目中需要复用这些代码时不想将这些代码`CTRL+C`、`CTRL+V`，与C语言中的那些按功能划分的`.h`、`.c`文件类似，Python也提供了**模块**和**包**用来提高代码复用率以及降低代码规模。    
　　**模块**就是一个Python(以.py为后缀)文件，在模块中可以定义类、函数以及变量。通过import导入就可以在你写的代码中使用模块中的内容。   
## 模块的导入
　　除了import module这种用法之外，还有其他的一些使用方法，比如：`from module import *`，这种导入方法把模块（module）的所有内容全都导入，再比如：`from module import name`，这种用法可以让你从模块（module）中导入一个指定的部分（name）到当前命名空间中，使用from…import…语句可以避免在程序中使用前缀符，比如说，`from module import name`时，当需要使用name时，不需要通过module.name这种方式，只需要直接使用name就可以了。    
## 模块的属性
　　使用dir函数可以列出模块定义的标识符，标识符有函数、类和变量等等，以copy模块为例：    

	>>> dir(copy)
	['Error', 'PyStringMap', '_EmptyClass', '__all__', '__builtins__', '__doc__', '__file__', '__name__', '__package__', '_copy_dispatch', '_copy_immutable', '_copy_inst', '_copy_with_constructor', '_copy_with_copy_method', '_deepcopy_atomic','_deepcopy_dict', '_deepcopy_dispatch', '_deepcopy_inst', '_deepcopy_list', '_deepcopy_method', '_deepcopy_tuple', '_keep_alive', '_reconstruct', '_test', 'copy', 'deepcopy', 'dispatch_table', 'error', 'name', 't', 'weakref']
	>>> copy.__all__
	['Error', 'copy', 'deepcopy']
　　`__all__`属性定义了模块的公有接口，当你使用`from module import *`时你只能使用`__all__`变量中的几个成员，当使用`import module`时，便可以获得模块中所有不以下划线开头的属性。模块中可能会有许多其他程序根本不会需要的属性，`__all__`会把它们过滤掉。  
　　help函数和`__doc__`属性用于获得帮助的文档：help(copy)和`copy.__doc__`。　　  
　　`__file__`属性可以输出模块的源代码位置。   
## 系统路径
　　首先你需要知道平时我们使用的模块都是在哪些位置，换句话说，哪些文件夹中的代码可以被import，通过下面的代码可以打印出系统路径，也就是说在`sys.path`中的模块都是可以被import的。 
{% highlight python %} 
>>>import sys
>>>print sys.path
['', 'C:\\Windows\\system32\\python27.zip', 'C:\\Python27\\DLLs', 'C:\\Python27\\lib', 'C:\\Python27\\lib\\plat-win', 'C:\\Python27\\lib\\lib-tk', 'C:\\Python27', 'C:\\Python27\\lib\\site-packages']
{% endhighlight %} 
## 模块的搜索路径
　　在导入模块时模块的搜索路径就是按照`sys.path`中的顺序（上一节系统路径中打印出来的顺序），首先是空字符串，空字符串代表代表当前目录（如果是在脚本中打印出来的话，可以更清楚地看出是哪个目录），接下来是Python本身的那些库的位置。  
## 使用自己编写的模块　　
　　假设你从之前的项目中抽取了一些基本的功能编写成模块（假设名字为ToBeCalled.py），当你在新的项目（名为ProjectNow）中的call.py需要使用它时，你可以复制ToBeCalled.py到你当前项目的文件夹中，目录结构如下：     

	ProjectNow
		-call.py
		-ToBeCalled.py
　　在call.py中你通过`import ToBeCalled`就可以使用ToBeCalled模块中的类或者函数了。这种方法每当你需要在新的项目中使用ToBeCalled模块时你都需要拷贝一份到新的项目中，而且当你修改了ToBeCalled中的bug之后，所有它的备份都需要被修改，维护的难度比较大。所以这种做法不是特别推荐。  
　　使用这种方法时有个需要注意的问题是当你在命名你的模块时尽量不要和标准的模块重名，因为模块的搜索路径中当前目录的优先级是最高的，所以会覆盖标准的模块，你可以通过`<module>.__file__`获取你导入的包的真实路径。        
　　比较好的做法就是你将ToBeCalled模块拷贝到系统路径中，这样在你所有的项目中都可以import ToBeCalled，而且在后续的维护中也比较简单。  
## 包
　　包是由一系列模块组成的集合，模块是处理某一类问题的函数和类的集合，`包必须至少含有一个__int__.py文件，内容可以为空，__int__.py标识当前文件夹是一个包`，`__int__.py`其实就是一个普通的python文件，它会在包被导入时执行，比如说（前提是packagename在系统路径中）：    
  
	packagename
		-__int__.py
		-module1.py
		-module2.py
　　在上面的例子中，packagename是一个包，因为在它里面包含了`__int__.py`文件，下面列出使用import导入包中的模块时的方法，假如`__int__.py`为空，那么仅仅导入包是没什么用的；如果在`__init_.py`中导入了module1的话，那么`import packagename`之后module1是可用的（`通过packagename.module1使用`）。  

	import packagename              # __init__中的内容为空时，module1和module2模块不可用
	import packagename.module1      # module1可用，但只能使用packagename.module1来使用
	from packagename import module2	# module2可用，可通过module2直接使用
　　在上面的例子中可以看到，包在使用import时可以通过`点（.）`来访问，比如`import packagename.module1`与`from packagename import module1`功能是一样的，区别还是在于访问时是否可以省去包名直接使用模块名进行访问。    