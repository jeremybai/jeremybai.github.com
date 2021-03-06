---
layout: post
title: "K60驱动步进电机中遇到的光电隔离问题"
description: ""
categories: 
- K60
- 电路
tags: [光电隔离]
---

　　光电隔离这个概念之前听说过，但是也是仅限于听说过，自己并未遇到过需要隔离的情况。最近在用K60驱动步进电机运动的时候就遇到了点问题，最终将问题定位到了光电隔离这一块，搞了几天才搞定，果然遇到问题才是学习知识最有效的途径。
## 问题描述 ##
　　问题是这样产生的：我手上有一个42的两相步进电机，一个M880A步进电机驱动器，还有一块K60开发板，当我想用K60来驱动步进电机运动的时候，首先想到了使用FTM模块输出PWM脉冲输出给驱动器来驱动步进电机，于是我直接将K60的PWM输出引脚接到驱动器上，再加上2个GPIO口控制步进电机的方向和使能信号，连线完成之后将步进电机驱动起来了！（是不是很简单，可是噩梦还没开始！！！）因为步进电机上滑块的运动是有范围的，不能超过滚珠丝杠的长度，于是我将步进电机两端各装了一个磁簧开关，使用GPIO口中断来检测磁簧开关的状态，磁簧开关的原理图如下。  
[![磁簧开关原理](http://github-blog.qiniudn.com/2014-08-30-optocoupler-1.png-BlogPic)  ](http://github-blog.qiniudn.com/2014-08-30-optocoupler-1.png)  
　　将棕色线接GPIO引脚，引脚设置为输入，上拉，下降沿产生中断。问题来了，单独测试步进电机和磁簧开关的时候，实验结果让人很欣慰，可是当我将步进电机上电之后测试发现磁簧开关对应的GPIO中断不停的发生，用示波器测量了下棕色引脚的波形，对比电机不上电时的波形，发现多了许多毛刺。既然问题出在电机是否上电，想到有可能2个原因：1 磁场收到干扰，磁簧开关受到影响；2 电机对单片机形成干扰，需要光电隔离。之所以将光电隔离放在第二位，因为M880A驱动器手册上写了其内部已经做了光电隔离。第一个原因是因为我的步进电机装在一个钢条做成的四边形上，于是怀疑是不是上电之后环形的钢条形成磁场对开关有干扰，于是找了个金属外壳将K60开发板放入其中，可是发现问题还在，后来便想到应该驱动器的光电隔离做的并不好，于是便准备自己搭一个光电隔离模块看看效果怎么样。  
## 解决过程 ##
　　在实验室找了几个8050，光耦只找到MOC3022，不是高速光耦，不过我的PWM频率只有1KHz，应该也不算高速，于是就先用看看。搭建电路图如下：    
[![MOC3022电路](http://github-blog.qiniudn.com/2014-08-30-optocoupler-2.png-BlogPic)  ](http://github-blog.qiniudn.com/2014-08-30-optocoupler-2.png)    
　　测试之后，发现输出的波形不太好，下降沿是曲线，用这个波形驱动电机驱动不起来，于是想着是不是响应频率不够，换个速光耦试试，于是托人从赛格带了几个6N138，于是又换上6N138，电路如下：   
[![6N138电路](http://github-blog.qiniudn.com/2014-08-30-optocoupler-3.png-BlogPic)](http://github-blog.qiniudn.com/2014-08-30-optocoupler-3.png)  
　　6N138手册里写道：*电源管脚旁应有—个0.1uF的去耦电容。在选择电容类型时，应尽量选择高频特性好的电容器，如陶瓷电容或钽电容，并且尽量靠近6N137光耦合器的电源管脚；另外，输入使能管脚在芯片内部已有上拉电阻，无需再外接上拉电阻。。。6N137光耦合器的使用需要注意两点：第一是6N137光耦合器的第6脚Vo输出电路属于集电极开路电路，必须上拉一个电阻；第二是6N137光耦合器的第2脚和第3脚之间是一个LED，必须串接一个限流电阻。*  
　　完成之后测试发现6N138的输出波形下降沿很正，但是上升沿不是很好，弯着上去的，我试着驱动电机看，发现是可以驱动电机的，于是又喊了两路，用于隔离驱动器的使能和方向，测试ok，再加上磁簧开关，发现频繁的中断消失了，只有在磁铁靠近磁簧开关的时候才会进入中断，终于搞定了！


