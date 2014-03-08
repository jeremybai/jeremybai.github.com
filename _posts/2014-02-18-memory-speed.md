---
layout: post
title: "memory的访问速度比较"
description: ""
categories: 
- 计算机组成
tags: []
---
{% include JB/setup %}

　　首先上一张图，图中各个部件的价格可能和现在差距比较大，但是我们主要关注的是访问速度。  
![](http://image.beekka.com/blog/201310/2013101401.png)  
　　我们知道，**cpu只能和register（寄存器）进行数据的交互**，寄存的意思就是是暂时存放数据，不用每次从内存中读取，是一个临时放数据的空间。寄存器中的数据来自于ram(内存)。所以信息之间的交换为：cpu<-->register<-->ram。  

　　寄存器

　　但是因为ram访问速度不是很快，每次都要从ram中取数据会严重的影响效率，所以在register和ram之间加了一个cache(缓存)，cache就把从ram里面取出的数据暂时保存在里面，如果register要取ram中同一位置的东西，就不用大老远的跑到内存中去取，直接从cache中取就可以了(之前的《const volatile修饰的变量》里面说的就是这个道理，volatile修饰的作用就是每次读取数据不做优化，直接从内存的指定地址读取数据，虽然慢点，但是在特定情况下保证了数据的准确)。但是，register并不每次数据都可以从cache中取得数据，万一需要访问内存地址的数据不在cache中，那register还必须直接绕过cache从内存中取数据。所以并不每次到cache中取数据都是成功的，这就是**cache的命中率**，数据在cache中就命中，不在就没命中。此时信息之间的交换为：CPU<-->register<-->cache<-->ram。cache又分为L1 cache（一级缓存）和L2 cache（二级缓存）等等。对于各级的cache，访问速度是不同的，理论上说L1 cache有着跟CPU寄存器相同的速度，但L1 cache有一个问题，当需要同步cache和内存之间的内容时，需要锁住cache的某一块（术语是cache line），然后再进行cache或者内存内容的更新，这段期间这个cache块是不能被访问的，所以L1 cache的速度就没寄存器快，因为它会频繁的有一段时间不可用。L1 cache下面是L2 cache，甚至L3 cache，这些都有跟L1 cache一样的问题，要加锁，同步，并且L2比L1慢，L3比L2慢，这样速度也就更低了。

　　从制作材料看，SRAM（Static Random Accessible Memory）的1bit由6个晶体管组成，一般作为片上存储器使用（On-chip-memory）。它消耗的晶体管比较多，但是速度快，所以用来制作寄存器和缓存。DRAM（Dynamic Random Accessible Memory）的1bit由一个晶体加一个电容组成。虽然面积小，但是速度慢，而且需要刷新，所以通常用来制作内存。

　　从计算机体系结构角度而言，需要把不同速度和容量的memory分层级，得到**速度**、**容量**和**成本**间较好的平衡。最需要经常访问的数据放在速度最快容量最小价格最高的寄存器和L1 cache里，访问量最少的数据放在最慢最大价格相对最低的内存条里，以此类推。

