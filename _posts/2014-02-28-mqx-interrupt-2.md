---
layout: post
title: "MQX机制分析——中断机制(二)"
description: ""
categories: 
- C
tags: [MQX,操作系统,飞思卡尔]
---

　　之前介绍了MQX中断机制的主要特点以及向量表的特别之处，这一节具体分析一些细节。
## 中断优先级 ##
　　首先简单介绍下Cortex M4的优先级的异常响应，Cortex M4支持最多255个中断（0代表没有异常在运行），其中中断号1-15对应系统异常，中断号16到255为外部中断（这里的中断号指的是NVIC所使用的中断号）。优先级值越小优先级越高，除了前三个中断Reset（复位）、NMI（不可屏蔽中断）、HardFault（硬fault）之外（分别是-3～-1），其他都可以用户自己定义。优先级配置寄存器有8位，厂商可以根据自己的芯片的用途进行裁剪，K60使用4位表示优先级，MQX中使用了高3位（范围0-7）来表示优先级。
	
    #define CORTEX_PRIOR_IMPL   (3)

　　中断的优先级是由NVIC_IPR0～NVIC_IPR59这60个32位的寄存器控制的，每个32位的寄存器被分成4个8位的寄存器用来表示每个等级的中断优先级（抢占优先级和亚优先级这里不做说明，可以参考《ARM Cortex-M3权威指南》），以IPR0为例（下图是从K60芯片手册截图）：  
  
![1](http://github-blog.qiniudn.com/2014-02-28-mqx-interrupt-2-1.png-BlogPic)
　　我们可以计算一下：0-59这60个32位寄存器可以控制的中断等级为32/8*60=240个，正好等于255-15。而前15个中断优先级（前三个不能被修改）则是由SHPR1-SHPR3（System Handler Priority Register，系统异常优先级寄存器）控制。   

　　和异常相关的寄存器有：PRIMASK、FAULTMASK和BASEPRI。PRIMASK屏蔽所有外部中断，只有NMI和硬fault可以被响应；FAULTMASK和PRIMASK工作方式相同，区别在于FAULTMASK屏蔽除NMI外其他所有中断，包括硬fault;而BASEPRI代表一个阈值，只有比这个优先级高的异常才能响应（也就是值小于BASEPRI寄存器的值）。而关于单个中断的开关是由NVIC\_ISER0-7（中断使能寄存器）和NVIC\_ICER0-7（中断除能寄存器）控制的。程序里经常使用的CPSID I和CPSIE I指令设置的是PRIMASK寄存器，而CPSID F和CPSIE F设置的是FAULTMASK寄存器。  
## 任务优先级和中断优先级 ##
　　在MQX中，**中断的触发会受到当前运行的任务的影响**（后面会讲到如何影响）。任务的优先级和中断的优先级是绑定的，任务的优先级范围理论上是0-65535，MQX中使用了高3位（0-7）来表示中断优先级，所以中断的优先级范围是0-7。这里还要介绍几个概念：ENABLE\_SR、TASK\_SR、ACTIVE\_SR和DISABLE\_SR。这几个变量存在于不同的结构体中，有些是全局的，有些是任务独享的，不仅和中断有关，还和任务调度有关。看代码时被这几个概念弄得“意识模糊”了。重新看代码看了好几遍才稍微清楚一点，下面这张图介绍了这几个变量的关系。    
![2](http://github-blog.qiniudn.com/2014-02-28-mqx-interrupt-2-2.png-BlogPic)
### ENABLE\_SR ###
　　ENABLE\_SR是在\_psp\_init\_readyqs(初始化就绪队列)函数中被初始化的：  
{% highlight c++ %}
...
n = priority_levels;//n代表就绪队列的数目
while (n--) 
{  
      q_ptr->HEAD_READY_Q  = (TD_STRUCT_PTR)q_ptr;
      q_ptr->TAIL_READY_Q  = (TD_STRUCT_PTR)q_ptr;
      q_ptr->PRIORITY      = (uint_16)n;

      if (n + kernel_data->INIT.MQX_HARDWARE_INTERRUPT_LEVEL_MAX < (1 << CORTEX_PRIOR_IMPL))
        q_ptr->ENABLE_SR   = CORTEX_PRIOR(n + kernel_data->INIT.MQX_HARDWARE_INTERRUPT_LEVEL_MAX);
      else
        q_ptr->ENABLE_SR   = 0;

      q_ptr->NEXT_Q        = kernel_data->READY_Q_LIST;
      kernel_data->READY_Q_LIST = q_ptr++;
}
...
{% endhighlight %} 
　　可以看出，对于不同的就绪队列，它们的ENABLE\_SR的值也是不一样的，CORTEX\_PRIOR这个宏将任务优先级加上MQX\_HARDWARE\_INTERRUPT\_LEVEL\_MAX（初始化结构体中设置为2，可以根据不同应用场合修改）作为参数计算出来（这个宏的作用就是将n+2左移5位）赋值给ENABLE\_SR。因为硬件优先级用了3位表示，也就意味着当任务的优先级大于等于6（6+2>7）的时候，已经不能对中断进行屏蔽了，所以优先级大于等于6的就绪队列的ENABLE\_SR都是0x00（之后把这个值赋给BASEPRI寄存器的时候就不能屏蔽任何中断了）。不同的n对于不同的ENABLE\_SR，当n大于等于6时便全部为0了。具体值如下所示，当MQX\_HARDWARE\_INTERRUPT\_LEVEL\_MAX被设置为2时，也就意味着任务可以响应比它优先级低1级的中断，比如说任务优先级为3，它所属的就绪队列的ENABLE\_SR等于0xA0，高3位等于5。那么当它运行的时候，中断优先级大于等于5（3+2）的中断都会被屏蔽掉，如果任务优先级为7，那么它运行的时候不能屏蔽任何中断。
![3](http://github-blog.qiniudn.com/2014-02-28-mqx-interrupt-2-3.png-BlogPic)
### TASK\_SR ###
　　每一个任务描述符结构体都有一个TASK\_SR成员，每当初始化创建任务的时候就会将就绪队列的ENABLE\_SR赋值给任务描述符的TASK\_SR，这也就意味着每个任务的TASK\_SR都是对应自己就绪队列的ENABLE\_SR。  

    td_ptr->TASK_SR = ready_q_ptr->ENABLE_SR;
### ACTIVE\_SR ###
　　ACTIVE\_SR是内核数据结构的一个成员变量，所以它是一个全局的变量，它在初始化内核数据区函数（\_mqx\_init_kernel\_data\_internal）中被赋值为DISABLE\_SR的值：  

    kernel_data->ACTIVE_SR = kernel_data->DISABLE_SR;
　　每当一个任务被置于激活态时（即将开始运行）它都会被赋值为即将执行的任务的TASK\_SR，也就是该任务所属的就绪队列的ENABLE\_SR，我们在dispatch.s中可以看到：
{% highlight html %}　
ASM_LABEL(switch_task)
                str r1, [r0, #KD_CURRENT_READY_Q]   /* 把当前就绪队列的地址存入内核数据区对应的位置*/
                str r2, [r0, #KD_ACTIVE_PTR]        /* 把当前激活任务的就绪队列存入内核数据区对应的位置,r2存放的是激活任务*/
                /* 将激活任务的TASK_SR赋值给内核数据区的ACTIVE_SR */
                ldrh r3, [r2, #TD_TASK_SR]			
                strh r3, [r0, #KD_ACTIVE_SR]        
{% endhighlight %} 
### DISABLE\_SR ###
　　DISABLE\_SR和ACTIVE\_SR类似，都是存在于内核数据区的，它的初始化是在\_psp\_set\_kernel\_disable\_level函数中设置的:
{% highlight c++ %} 
//MQX_HARDWARE_INTERRUPT_LEVEL_MAX=2
temp = init_ptr->MQX_HARDWARE_INTERRUPT_LEVEL_MAX;
    if (temp > 7) {
        temp = 7;
        init_ptr->MQX_HARDWARE_INTERRUPT_LEVEL_MAX = 7;
    } else if (temp == 0) {
        temp = 1;
        init_ptr->MQX_HARDWARE_INTERRUPT_LEVEL_MAX = 1;
    }
    kernel_data->DISABLE_SR = CORTEX_PRIOR(temp); //DISABLE_SR = 0X40
{% endhighlight %} 
　　再一次看到这个CORTEX_PRIOR宏了，可以计算得出DISABLE\_SR被初始化为了0x40。  

　　我们重新再把这几个变量捋一遍，首先在初始化的时候，最先被赋值的是全局的DISABLE\_SR，它被赋值为了0x40；接着在初始化内核数据区时会将DISABLE\_SR赋值给了全局的ACTIVE\_SR，这样ACTIVE\_SR也等于0x40；创建就绪队列的时候，会给每一个就绪队列一个ENABLE\_SR；只要任务被创建时被放到哪个就绪队列，它所在的就绪队列的ENABLE\_SR的值就会赋给该任务的TASK\_SR；当这个任务被置于激活态时，也就是即将运行的时候，任务的TASK\_SR就会被赋给全局的ACTIVE\_SR。  
### 关中断开中断 ###
　　说到底，最终有影响力的还是这两个全局的变量：DISABLE\_SR和ACTIVE\_SR，因为他们两个是赋值的“终点”。从另外一个方面考虑，在MQX中的关中断和开中断并不是像其他的RTOS里面可能只是简单的CPSIE I和CPSID I两条指令，而是使用的\_int\_disable和\_int\_enable这两个函数。我们说过MQX中正在运行的任务会对当前触发的中断产生影响，从刚才这几个变量赋值的流程中，是不是能够略微嗅出什么呢？这里就要介绍MQX的关开中断了，首先介绍关中断的函数：  
{% highlight c++ %} 
#define _INT_DISABLE_CODE()                             \
   if (kernel_data->ACTIVE_PTR->DISABLED_LEVEL == 0)    \
   {                                                    \
      _PSP_SET_DISABLE_SR(kernel_data->DISABLE_SR); /*修改basepri*/ \   
   }                                                          \
   ++kernel_data->ACTIVE_PTR->DISABLED_LEVEL;
{% endhighlight %}  
　　在关中断的时候，首先判断DISABLED\_LEVEL是否为0，这个变量表示的是当前运行任务运行时\_int\_disable被调用了多少次。\_PSP\_SET\_DISABLE\_SR作用就是将参数赋值给BASEPRI寄存器，如果是第一次调用就将DISABLE_SR赋值给BASEPRI；如果不是第一次调用，则将DISABLED\_LEVEL加1。这也就意味着调用\_int\_disable（\_int\_disable = 0x40）之后，所以优先级的值大于等于2的中断都将得不到响应（高3位表示优先级）。 
{% highlight c++ %} 
#define _INT_ENABLE_CODE()                                  \
   if (kernel_data->ACTIVE_PTR->DISABLED_LEVEL) {           \
      if (--kernel_data->ACTIVE_PTR->DISABLED_LEVEL == 0) { \
         if (kernel_data->IN_ISR) {  /*存在中断嵌套的时候*/   \
            _PSP_SET_ENABLE_SR(kernel_data->INTERRUPT_CONTEXT_PTR->ENABLE_SR); /*修改basepri*/ \
         } else {                                           \
            _PSP_SET_ENABLE_SR(kernel_data->ACTIVE_SR);     \
         }                                                  \
      }                                                     \
   } 
{% endhighlight %} 
　　开中断相比于关中断稍微复杂一点，首先判断DISABLED\_LEVEL是否为0，也就是是否调用过\_int\_disable，没有的话什么都不做；如果调用过一次，将DISABLED\_LEVEL减1就可以了；如果不止调用了一次，再判断IN_ISR的值，IN_ISR是内核数据的成员，它表明了中断嵌套的层数， 如果不存在中断嵌套，就将ACTIVE\_SR赋值给BASEPRI寄存器（\_PSP\_SET\_ENABLE\_SR和\_PSP\_SET\_DISABLE\_SR最后都被定义为给BASEPRI寄存器赋值）。举个例子，比如一个任务的优先级是3，那么它会被放在ENABLE\_SR等于0xA0的就绪队列中，它的TASK\_SR也就等于0xA0，当它运行时就会将它的TASK\_SR（0xA0）赋值给ACTIVE\_SR，这意味着该任务运行时，优先级的值大于等于5的中断都将得不到响应；如果存在中断嵌套，就将kernel\_data->INTERRUPT\_CONTEXT\_PTR->ENABLE\_SR赋值给BASEPRI寄存器，看到这里有人可能就疑惑了，ENABLE\_SR不是就绪队列的成员吗，怎么又会在这里出现？不知道是因为什么原因，在中断上下文结构体中也有一个ENABLE\_SR成员，它和我们之前介绍的就绪队列结构体中的ENABLE\_SR没有关系。INTERRUPT\_CONTEXT\_PTR结构体是用来记录中断相关信息的，比如中断号、错误码等等：     
{% highlight c++ %} 
typedef struct psp_int_context_struct
{
    /* 前一个INT_CONTEXT结构体，如果没有就为NULL*/
    struct psp_int_context_struct _PTR_ PREV_CONTEXT;
    /* 中断异常号*/
    uint_32                             EXCEPTION_NUMBER;
    /* 供ISR中的_int_enable使用*/
    uint_32                             ENABLE_SR;
    /* 错误码*/
    uint_32                             ERROR_CODE;
} PSP_INT_CONTEXT_STRUCT, _PTR_ PSP_INT_CONTEXT_STRUCT_PTR;
{% endhighlight %}
　　这些上下文结构体是由中断栈中的一个链表维护的，当中断发生时首先进入\_int\_kernel\_isr，会将IN_ISR（中断嵌套层数）加1，接着将4个变量入栈，分别是：错误码、BASEPRI、IPSR（异常号）以及内核数据区的当前中断上下文压入中断栈，正好就对应于PSP\_INT\_CONTEXT\_STRUCT结构体的四个成员，**这样一来就相当于给INTERRUPT\_CONTEXT\_PTR赋了值**，压完之后会将当前的MSP存到INTERRUPT\_CONTEXT\_PTR中去，这样一来，**每次发生中断的时候INTERRUPT\_CONTEXT\_PTR都是指向的上一次中断的相关信息了**。下图显示了中断栈中这些中断上下文结构体如何存储的。  
![3](http://github-blog.qiniudn.com/2014-02-28-mqx-interrupt-2-4.png-BlogPic)  
　　对于的压栈代码在dispatch.s的\_int\_kernel\_isr中，片段如下：  
{% highlight c++ %} 
//按PSP_INT_CONTEXT_STRUCT结构存储上文
ldr r0, =0     /* 错误代码（0对应于MQX_OK） */
push {r0}      /* 将错误代码存入栈中 */
mrs r2, BASEPRI/* 读取本次BASEPRI寄存器中的值存至R2寄存器中 */
mrs r1, IPSR   /* 寄存器中的中断（当前正在运行的）向量号至r1寄存器中 */
// 获取内核数据区中INTERRUPT_CONTEXT_PTR（PSP_INT_CONTEXT_STRUCT结构体类型的变量)把上次中断MSP的首地址存至r0中
ldr r0, [r3, #KD_INTERRUPT_CONTEXT_PTR]  
//将r0-r2的值入栈（r0：上次中断内容首地址；r1：当前的中断向量号；r2：当前中断BASEPRI的值）
push {r0-r2}                    
//保存本次MSP于kernel data的INTERRUPT_CONTEXT_PTR
mrs r0, MSP                     
str r0, [r3, #KD_INTERRUPT_CONTEXT_PTR] 
{% endhighlight %}   
 　　接着之前的开中断讲，我们说为什么当存在中断嵌套的时候要将kernel\_data->INTERRUPT\_CONTEXT\_PTR->ENABLE\_SR赋值给BASEPRI寄存器呢？有可能是为了防止BASEPRI被修改，因为当存在中断嵌套的时候，获取之前发生的中断的相关信息如中断号，优先级什么的就会比较困难，INTERRUPT\_CONTEXT\_PTR的作用就体现出来了，想要之前被中断的中断信息直接找它就好了，因为在发生中断的时候，我已经将这些信息都以链表的形式存好了。举个例子，在中断2的时候，发生了一个更高的中断3，在中断3中修改了BASEPRI寄存器，当我再想恢复BASEPRI的时候，不应该将它恢复成之前被中断的任务的ACTIVE\_SR，而是之前被中断3中断的中断2的BASEPRI寄存器的值，也就对应于INTERRUPT\_CONTEXT\_PTR->ENABLE\_SR了。
## 小结 ##
 　　真正将任务的优先级和中断的优先级联系起来的起始就是\_int\_disable和\_int\_enable这两个函数了，通过改变BASEPRI寄存器的值从而达到运行不同任务的时候根据任务的优先级动态的修改能够响应的中断的优先级。这也就是在前一节的开头介绍的MQX五个特点时的第五个特点：任务优先级和中断优先级相联系。在我们的印象中一直有这样一个认识：当中断发生时，无论你在做什么任务，都要停下来去响应它。但是这样做会有一个缺点，就是当执行的任务优先级很高时，并不希望被低等级的中断所打断时，中断立即响应的这个优点反而变成了缺点。MQX采用的将中断优先级与任务优先级相绑定的方法使得一个高优先级的任务不会被一些低优先级的中断所打断。
## 问题 ##
 　　1 BASEPRI寄存器只有在调用\_int\_disable和\_int\_enable这两个函数的时候才会被改变，当一个任务被转为就绪态开始执行的时候，ACTIVE\_SR被改变了，但是BASEPRI没变，当我没有调用\_int\_enable时，BASEPRI就不可能被修改为ACTIVE\_SR，也就是任务的TASK_SR，是不是就达不到根据运行的任务动态的修改能够响应的中断优先级的目的？还是说在任务即将运行时BASEPRI就改变了，只是我没找到代码而已。

 　　<!--讲到现在将MQX的中断机制简要的介绍了一遍，这里将我觉得理解上可能会有问题的地方介绍了，可能还有很多地方没有讲到，毕竟MQX的中断机制还是蛮复杂的，但是复杂也使得MQX更加灵活。-->