---
layout: post
title: "x86中函数到底是如何被调用的"
description: ""
categories: 
- c
- 汇编
tags: []
---

　　在之前的《MQX机制分析——中断机制(三)》这篇文章中有提到过中断响应的流程，其实中断函数的执行和普通函数的调用过程存在一些类似的地方，都需要将当前程序状态进行保存，然后才能跳转至函数执行，只不过中断函数在保存当前程序状态相关的寄存器之前，也就是中断发生的时候由中断控制器NVIC获取中断源硬件设备的中断向量号，找到对应的中断函数进行执行。不过在x86中的函数调用与ARM Cortex-M4中还是有些差异的，下面在x86架构下看下函数调用到底是如何实现的。
## 大致流程 ##
　　我们首先从整体上思考下函数调用的时候需要做些什么，以下面一段代码为例，这个例子会贯穿这篇文章。
{% highlight c++ %}
#include <stdio.h>

int add(int a, int b) {
    int sum = 0;
    sum = a + b;
    return sum;
}

int main(int argc, char *argv[]) {
    int sum = 0;
    sum = add(1, 2);
    printf("result = %d \r\n", sum);
    return 0;
}
{% endhighlight %} 
　　函数存储在内存中某一块区域（代码段），函数的调用其实就是程序跳转到另外一块区域执行的过程，那么既然跳过去了，那应该就需要再跳回来，那么就需要保存调用完函数之后的下一条指令的地址；另外当调用函数有参数的时候需要将参数进行传递，所以还需要保存参数的值；剩下就是一些寄存器的值，因为在函数执行中，我们需要用到他们，再返回之后，这些寄存器应该恢复到调用函数之前的状态。主要需要完成的工作也就是上面列举的三部分了。在ARM Cortex-M4架构中，有链接寄存器（LR）专门用于存放函数的返回地址[3]。而在x86中这三部分在x86中都是通过栈来实现的，栈是一种特殊的数据结构，它的功能是让数据后进先出，x86中栈的地址是向下增长的，即从大地址向小地址生长，那么压栈push的操作会使得栈顶指针减小，出栈pop操作会使得栈顶指针增大。函数调用时维护的栈我们将它称作“堆栈帧”，它包括上面列举的三部分[1]：  
（1）函数的返回地址和参数  
（2）临时变量，比如函数的非静态的局部变量  
（3）保存的上下文，也就是在函数调用前后需要保持不变的寄存器  
　　在x86中，堆栈帧是由ebp和esp这两个寄存器来划定范围的，esp指向的是栈的顶部，也是堆栈帧的顶部，ebp指向指向了一个固定的位置，常见的堆栈帧的结构如下：  
[![堆栈帧结构](http://github-blog.qiniudn.com/2014-08-12-function-call-1.png-BlogPic) ](http://github-blog.qiniudn.com/2014-08-12-function-call-1.png)  
　　下面我们将中断代码进行反汇编查看在汇编代码中函数调用时如何实现的。  
## 反汇编
　　找到`sum = add(1, 2);`这段代码对应的汇编代码我们逐句看一下：  
{% highlight ca65 %}
    sum = add(1, 2);
011C1405  push        2  
011C1407  push        1  
011C1409  call        add (11C108Ch)  
011C140E  add         esp,8  
011C1411  mov         dword ptr [sum],eax  
{% endhighlight %} 
　　从汇编代码可以看出，程序首先将函数参数进行入栈，并且是按照从右到左的顺序，然后就通过call指令跳转到add函数的入口，call指令有两个功能：（1）将当前指令的下一条指令的值（ip的值）压入堆栈；（2）跳转到被调用函数。我们继续反汇编查看add函数对应的汇编代码：  
{% highlight ca65 %}
int add(int a, int b) {
011C1390  push        ebp  
011C1391  mov         ebp,esp  
011C1393  sub         esp,0CCh  
011C1399  push        ebx  
011C139A  push        esi  
011C139B  push        edi  
011C139C  lea         edi,[ebp-0CCh]  
011C13A2  mov         ecx,33h  
011C13A7  mov         eax,0CCCCCCCCh  
011C13AC  rep stos    dword ptr es:[edi]  
    int sum = 0;
011C13AE  mov         dword ptr [sum],0  
    sum = a + b;
011C13B5  mov         eax,dword ptr [a]  
011C13B8  add         eax,dword ptr [b]  
011C13BB  mov         dword ptr [sum],eax  
    return sum;
011C13BE  mov         eax,dword ptr [sum]  
}
011C13C1  pop         edi  
011C13C2  pop         esi  
011C13C3  pop         ebx  
011C13C4  mov         esp,ebp  
011C13C6  pop         ebp  
011C13C7  ret  
{% endhighlight %}   
　　跳转到被调用函数之后并没有直接开始执行函数，而是首先将ebp寄存器入栈，这里是为了防止在函数中修改了ebp，ebp入栈之后就可以通过pop将ebp的值恢复，接着将esp赋值给了ebp，此时ebp就指向了栈中ebp存放的位置，这个位置是固定的（在函数执行过程当中），然后将esp减去0CCh，也就是将栈扩大了0CCh这么多个字节，这段空间用于存储一些需要保存的寄存器（接下来的ebx、esi、edi寄存器入栈）、局部变量以及某些临时数据和调试信息。011C139C到011C13AC这几句代码是将调试信息加入其中。  
　　`lea   edi,[ebp-0CCh]` 也就是ebp-0CCh这个地址（栈顶地址）放入edi中。这等同于：`mov edi,ebp-0cch`，但是以上mov指令是错误的，因为mov不支持后一个操作数写成寄存器减去数字，但是lea支持，所以可以用lea来代替它。再将ecx寄存器，也就是计数寄存器赋值为33h，eax寄存器赋值为0CCCCCCCCh，stos是串存储指令，它的功能是将eax中的数据放入edi所指的地址中，同时edi会增加4字节（操作符`X ptr`指明指令访问内存单元的长度，X在汇编指令中可以为dword、word或byte）。rep使指令重复执行ecx中填写的次数。方括弧表示存储器，这个地址实际上就是edi的内容所指向的地址。这里stos其实对应的是stosd，其它还有stosb、stosw,分别对应于处理4、1、2个字节，这里对堆栈中33h*4(0CCh)个字节初始化为0CCh(也就是`int 3`指令的机器码)，这样发生意外时执行堆栈里面的内容会引发调试中断。  
　　接着才开始执行函数体部分，当执行完成之后，将sum存放到eax中，此时函数执行完成，需要恢复调用前的状态，此时的操作与前面调用函数的顺序相反，也就是将edi、esi、ebx出栈，然后恢复esp，也就是将堆栈帧“抛弃”了。这时esp指向的就是之前的ebp，也就实现了恢复ebp的目的。接着通过ret命令回到调用函数的下一条语句，与call类似，ret指令做了两个操作：弹出返回地址到ip，再跳转至ip指向的语句，可以这样理解：call等于push+jmp，ret等于pop+jmp。  
　　返回第一段代码，当调用完add函数之后，将esp加8，对应前面的将参数2和1入栈，这里没有将2和1出栈，而是直接将参数抛弃。然后再将eax给了sum变量，得到了返回值。至此，函数调用就完成了。    
### 参考文献
　　[1] [程序员的自我修养 ](http://book.douban.com/subject/3652388/)  
　　[2] [关于EBP寄存器的作用](http://kuaile.in/archives/714)  
　　[3] [CPU体系架构-寄存器](http://nieyong.github.io/wiki_cpu/CPU%E4%BD%93%E7%B3%BB%E6%9E%B6%E6%9E%84-%E5%AF%84%E5%AD%98%E5%99%A8.html)  


