---
layout: post
title: "使用36-pin的STM32输出VGA"
description: ""
categories: 
- 翻译
tags: [VGA,嵌入式]
---
{% include JB/setup %}

　　手头上有个项目需要通过单片机来控制将图像显示在LCD上，在网上搜了一阵子，发现都是使用的FPGA做的，开始自己对FPGA不是很熟，一直在用的也是ARM系列的，终于让我找到一份至少现在看起来还是含金量蛮高的资料，因为是英文的，这边先将它翻译一下（[**原文链接**](http://www.artekit.eu/vga-output-using-a-36-pin-stm32/)）。  

----------

　　想到之前玩的一些老的视频游戏和街机游戏（很早之前，大概70/80年代左右），脑子里浮现出一个想法：如果在今天，我们是不是可以使用成本比较低的微控制器来实现之前玩玩的那些游戏呢？这些微控制器设计的初衷并不是用来干这些事情的，所以问题也就产生了：如何在使用很少或者不使用外部组件的情况下向显示器输出视频信号呢？

　　我们选择了36-pin, 72 MHz的STM32 (STM32F103T8U6),足够用于产生黑白视频信号和点信号，同时还使用了一些定时器和SPI（在这种方式下更新帧缓冲是自动完成的），在400*200分辨率的显示器上VGA输出视频信号看起来还是比较可观的。

### 　　使用的材料： ###  
　　1 STM32F103T8U6开发板一块（或者同类型的开发板）。我们使用的是AK-STM32-LKIT。  
　　2 VGA母口一个（DB15）

　　虽然帧缓冲区是400*200的，但是输出的分辨率却是800*600（56hz刷新频率），我们采用把横着点绘制两次，竖着的点绘制三次的方法来达到扩展分辨率的目的。  

　　我们选择800×600 @ 56Hz的原因是因为像素时钟；输出分辨率使用36MHz像素时钟，周期是72MHz的倍数（STM32的频率），因为我们需要使用SPI产生像素信号，可以把STM32的频率经过SPI预分频得到18MHz的像素时钟，然后将每一个像素点绘制两次，具体方法是当在水平方向800像素点时输出一个信号像素，SPI 的 MOSI信号保持低电平或者高电平两倍的时间（相比于之前绘制一个点的时间）。  
![VGA output on a Samsung 17″ LCD monitor](http://github-blog.qiniudn.com/2014-04-20-VGA-output-using-arm-1.png-BlogPic)
　　帧缓冲区是一个52×200字节的数组。每一行有50*8=400个像素（每一个bit是一个像素），剩下的两个字节（52-50）模拟每一行的消隐间隔。
{% highlight c++ %}
#define VID_VSIZE 200
#define VID_HSIZE 50
 
__align(4) u8 fb[VID_VSIZE][VID_HSIZE+2];
{% endhighlight %}
　　在这一块ram中写入的数据都会被输出到屏幕，DMA被设置为自动从数据缓冲区读取数据并且输出到SPI的MOSI引脚。
 ## 水平同步 ##
　　水平同步信号（ horizontal synchronism signal）和后延时间（back porch time）由TIM1定时器产生的通道1和2产生，TIM1定时器产生的通道1连接到PA8。
![HSYNC and HSYNC+BACKPORCH signals ](http://github-blog.qiniudn.com/2014-04-20-VGA-output-using-arm-2.png-BlogPic)  

　　H-SYNC也就是TIM1定时器的通道1将会产生水平同步信号给显示器。H-BACKPORCH也就是TIM1定时器的通道2,计算水平同步时间的和以及后延时间，这个定时器产生一个中断用于触发DMA开始通过SPI发送像素的请求。

　　帧缓冲里面的每一行都会重复这样的过程，
## 垂直同步 ##
　　TIM2定时器用于产生垂直同步信号，但是实在从机模式下。TIM2计算主机（TIM1）产生的H-SYNC脉冲数。  
![Timer 1 and Timer 2](http://github-blog.qiniudn.com/2014-04-20-VGA-output-using-arm-3.png-BlogPic)
　　TIM2的通道2通过PA1输出V-SYNC信号。  
　　TIM2的通道3将会触发一个中断当定时器的计数器达到V-SYNC的和垂直后沿时间。这个中断会设置一个变量表明正在扫描一个有效帧并且DMA可以开始发送像素到屏幕了。
![VSYNC and VSYNC+BACKPORCH signals ](http://github-blog.qiniudn.com/2014-04-20-VGA-output-using-arm-4.png-BlogPic)  
## 像素发生器 ##
　　像素由SPI的MOSI（PA7）产生。定时器TIM1的通道2产生一个中断用于使能DMA TX请求向SPI发送数据。DMA将会从帧缓冲区读取一行并且将数据放到SPI的DR寄存器。

　　DMA被设置用来在一行信号被发送之后产生一个中断，行号是递增的。因为我们将每一行发送了三次，我们在中断中将计数加1。当三行数据被发送出去，我们将DMA指针指向下一行的帧缓冲。  

　　当所有的行被发送出去，DMA被禁止知道下一个有效单的帧中断发生（TIM2通道3）。  
## 连接 ##
　　你只需要几根杜邦线和一个母口的VGA接口就可以完成这项工作了。  

　　VGA标准说输出信号应该在0.7V到1V之间，所以你需要在线上进行分压（串联68欧姆和33欧姆的电阻要比47pF和68欧姆的并联），我们已经测试了一系列的LCD寄存器在没有分压的情况下，工作起来还行。  

　　引脚发参考AK-STM32-LKIT扩展板接插件，引脚命名对于所有的STM32都有效。根据你自己所用的STM32的手册确定使用的引脚是否一致。
![Connections](http://github-blog.qiniudn.com/2014-04-20-VGA-output-using-arm-5.png-BlogPic)  
![Connections](http://github-blog.qiniudn.com/2014-04-20-VGA-output-using-arm-6.png-BlogPic)  
## 结论 ##
　　我们使用了一个低成本微处理器作为VGA控制器，实现这个目的的方法很多，但是这种方法不需要额外的组件除了一个VGA接口。

　　如果你使用的是更高级的STM32，你可以试着使用扩大缓冲区而且在DMA被禁止的时候写帧缓冲区以避免数据被割裂。

　　你可以下载[源码](http://www.artekit.eu/resources/blog/artekit_vga.zip)，在里面你可以找到画线描点等等的工具库。  

　　下一篇日志中会通过VGA样例来实现视频游戏。
　　