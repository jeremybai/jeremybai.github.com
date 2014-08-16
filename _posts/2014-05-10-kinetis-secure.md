---
layout: post
title: "如何将K60芯片加密"
description: ""
categories: 
- 嵌入式
tags: [嵌入式,K60,飞思卡尔]
---
{% include JB/setup %}

　　在将芯片做成产品时我们不得不考虑的一个问题就是如何防止flash中的代码被调试器读出来，前两天看到飞思卡尔的FAE jicheng0622的一篇关于[K60加密](http://blog.chinaaet.com/detail/33837)[1]的文章，于是便自己动手验证了下。

　　原理其实很简单，K60芯片手册第八章有说：在未加密状态，所有的flash指令都可以通过外部编程接口(JTAG和EzPort)执行；当flash处于加密状态(FTFL\_FSEC[SEC]字段等于00, 01, 或者11)时，编程接口只支持批量擦除（erase mass），不允许存储器的读写。所以我们要做的就是将FSEC[SEC]置为00,01,11就可以了，但是不像我们写普通寄存器那么简单，我们发现FTFL\_FSEC寄存器是只读寄存器，当复位时，MCU会从flash配置域中读取其中部分数据然后加载到FTFL\_FSEC中。那么flash配置域到底是什么呢？在芯片手册中我们看到图1这张表格。  
![1](http://github-blog.qiniudn.com/2014-05-10-kinetis-secure-1.PNG-BlogPic)  
　　flash配置域是在程序flash中的一块16字节的空间（从0x0\_0400-0x0\_040F），里面保存了默认的flash保护的设置，这些数据会在复位时加载到对应的FTFL寄存器中，为什么规定起始地址是0x0\_0400呢？因为CM3/4支持的最大的中断个数为0-255，也就是256个，中断向量表会从0地址开始存放，每个占4字节，所以256个中断最多占用的空间为256*4=1024=0x400，所以flash配置域必须要从0x400开始放，而不能小于0x400，不过这个也就是个猜测，不知道官方有没有这个说法，不管它了，我们感兴趣的是地址为0x0\_040C所存放的数据，这个字节的数据会被加载到FTFL\_FSEC寄存器中，所以我们现在的任务就是将这个地址写上01（00,01,11都行）。  
　　这里以苏州大学给出的K60例程为例，我们在其中vectors.c文件找到中断向量表的定义。
{% highlight c++ %}
const VECTOR vector_entry  __vector_table[] = //@ ".intvec" =
{
    VECTOR_000,           //初始化SP
    VECTOR_001,           //初始化PC
    VECTOR_002,
...
    VECTOR_255,
    CONFIG_1,
    CONFIG_2,
    CONFIG_3,
    CONFIG_4,
};
{% endhighlight %} 
　　紧接着这256个中断之后跟着的是4个CONFIG值，如果没错的话应该就是默认的flash配置域的值，跳转到定义，果然：
{% highlight c++ %}
#define CONFIG_1		(pointer*)0xffffffff 
#define CONFIG_2		(pointer*)0xffffffff 
#define CONFIG_3		(pointer*)0xffffffff
#define CONFIG_4		(pointer*)0xfffffffe
{% endhighlight %} 
　　ARM处理器默认是小端模式的，所以，我们要找的0x0\_040C也就对应着CONFIG_4的最低位的一个字节fe（0b11111110）,FTFL\_FSEC[SEC]字段等于10，也就是默认不加密，我们把它改成01，也就是fd，编译之后烧写到芯片中，复位之后芯片程序启动，如果此时我再想重新烧写程序时，就会弹出提示：  
![2](http://github-blog.qiniudn.com/2014-05-10-kinetis-secure-2.png-BlogPic) 
　　提示表明芯片以及被加密，如果想要解密的话就会触发内部flash进行mass erase，此时再烧写的话芯片内部的flash就以及被全部擦除了，你的代码也就没有了。

　　如果你的工程中中断向量表中没有这些配置值的话，你可以可以自己添加，我们找到链接文件，在其中可以看到以及定义了flash配置域的地址范围：
{% highlight c++ %}
m_cfmprotrom 	(rx) : ORIGIN = 0x00000400, LENGTH = 0x10
...
SECTIONS
{
...
  .cfmprotect :
  {
    . = ALIGN(4);
	KEEP(*(.cfmconfig))	/* Flash Configuration Field (FCF) */
	. = ALIGN(4);
  } > m_cfmprotrom
...
}
{% endhighlight %}
　　在SECTIONS中，我们看到所有目标文件中以.cfmconfig为后缀的section都会被放到m_cfmprotrom这个地址范围中。所以只需要在代码中加上下面一段：
{% highlight c++ %}
const char FlashConfig[16] __attribute__ ((section(".cfmconfig"))) = {
		0xff,0xff,0xff,0xff,
		0xff,0xff,0xff,0xff,
		0xff,0xff,0xff,0xff,
		0xfd,0xff,0xff,0xff,		
};
{% endhighlight %}
　　__attribute__ ((section(".cfmconfig")))的含义就是将这段代码链接之后生成的目标代码是放在.cfmconfig段中的，而所有.cfmconfig段的目标代码都会被放到m_cfmprotrom中，如果想确认下的话，可以打开生成的目标代码查看下：
![5](http://github-blog.qiniudn.com/2014-05-10-kinetis-secure-5.PNG-BlogPic)
　　图中可以看出，0x0\_040C地址确实被改成了fd。

----------

　　最后我要吐槽下,为什么飞思卡尔的OSJTAG的接口和J-Link转接板的JTAG的10pin接口不一样!!!，还要手动用杜邦线进行连接。OSJTAG接口如下[2]：    
![3](http://github-blog.qiniudn.com/2014-05-10-kinetis-secure-3.PNG)  
　　J-Link的JTAG接口如下：  
![4](http://github-blog.qiniudn.com/2014-05-10-kinetis-secure-4.PNG)    
　　如果你也遇到这种情况，对照两幅图接线就好了，其中OSJTAG中的EZP_CS在J-Link的JTAG转接板里面对应为nTRST(可以不接)，注意下就好了。


### 参考文献
　　[1] [http://blog.chinaaet.com/detail/33837](http://blog.chinaaet.com/detail/33837)  
　　[2] [http://andyhuzhill.github.io/embedded/2012/12/08/jtag-pins/](http://andyhuzhill.github.io/embedded/2012/12/08/jtag-pins/)