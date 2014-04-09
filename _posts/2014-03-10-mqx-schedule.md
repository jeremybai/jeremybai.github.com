---
layout: post
title: "MQX机制分析——调度机制"
description: ""
categories: 
- C
tags: [MQX,操作系统,飞思卡尔]
---
{% include JB/setup %}

　　多任务运行时，那么任务切换的时候就不可避免的要碰到任务调度这个问题了，这是操作系统的一个核心部分，	一般来说，调度和中断是有关联的，代码都是用汇编编写，MQX中也一样，与调度相关的代码在dispatch.s中可以找到。 


----------
 
　　在讲调度之前，首先我们要介绍两个中断：系统服务调用（Supervisor Call，SVC）和可挂起系统调用（Pendable Supervisor，PendSV），在其他的ARM处理器（比如ARM7）中有个被称为软件中断（SWI）的指令，SVC的地位和SWI是相同的，甚至机器码都是相同的，因为从CM3开始，异常的处理模型已经改变，所以该指令也被重新命名了。SVC中断通过执行SVC指令来产生。该指令需要通过一个立即数来充当指令代号。SVC异常服务例程（汇编编写）稍后会提取出此代号，从而获知本次调用的具体要求，在调用相应的服务函数。例如：`SVC   0x01`就是调用1号系统服务，那么在SVC指令执行完之后进行SVC中断处理函数（\_svc\_handler）的时候，会根据这个系统服务号来决定执行什么操作。系统服务号是由RTOS编写者决定，在MQX中主要的系统服务主要由执行调度（1），任务阻塞（2）以及任务切换（3）。

    ASM_EQUATE(SVC_RUN_SCHED, 1)
    ASM_EQUATE(SVC_TASK_BLOCK, 2)
    ASM_EQUATE(SVC_TASK_SWITCH, 3)
    ASM_EQUATE(SVC_MQX_FN, 0xaa)
　　在《MQX机制分析——中断机制(三)》的最后我们提到，一旦进入了用户级，那么如果想要再返回特权级，只能先触发一个软中断，再由中断服务例程修改该位。这样就可以返回特权级了，这里就可以使用SVC指令来触发软中断。这种机制使得户程序不需要直接和硬件打交道，而是由RTOS来管理硬件。并且使用户程序无需在特权级别下运行，用户程序不会因为误操作而使整个系统陷入混乱。  
　　另一个相关的异常时PendSVC，它和SVC合作使用。一方面，SVC异常时必须在执行SVC指令后立即得到响应的，应用程序执行SVC时都是希望所需的请求立即得到响应。另一方面，PendSV则不同，它可以像普通的中断一样可以被推迟执行，操作系统可利用它稍后执行一个异常（直到其他重要的任务完成后才执行动作）。推迟PendSV中断的方法是：往NVIC的PendSV挂起寄存器中写1。推迟后如果优先级不够高，则将等待执行。

----------
　　MQX中的SVC和PendSV中断的执行流程可以用下图来表示:  
[![1](http://a.hiphotos.bdimg.com/album/s%3D900%3Bq%3D90/sign=c2b9d78fbb12c8fcb0f3facdcc38e378/730e0cf3d7ca7bcbba746280bc096b63f624a816.jpg)](http://a.hiphotos.bdimg.com/album/s%3D900%3Bq%3D90/sign=c2b9d78fbb12c8fcb0f3facdcc38e378/730e0cf3d7ca7bcbba746280bc096b63f624a816.jpg)
　　在MQX中主要的系统服务主要由执行调度（SVC\_RUN\_SCHED），任务阻塞（SVC\_TASK\_BLOCK）以及任务切换（SVC\_TASK\_SWITCH）构成，SVC\_RUN\_SCHED 用于第一次启动调度器，SVC\_TASK\_BLOCK 用于将一个任务阻塞接着再执行调度，SVC\_TASK\_SWITCH 执行正常的任务调度。从图中我们可以看出 SVC\_RUN\_SCHED 和 SVC\_TASK\_SWITCH 灰色部分执行的操作是一致的，但是区别在于这一段代码在 SVC\_RUN\_SCHED 中是在SVC的中断服务例程（\_svc\_handler）中执行的，而在SVC\_TASK\_SWITCH中是在PendSV的中断服务例程结束中做的。还有一个区别在于SVC\_TASK\_SWITCH比SVC\_RUN\_SCHED多执行了一步，将{r2-r11, lr}压入任务堆栈。这要和任务栈的结构联系一起理解了，因为任务的切换实际上就是堆栈的切换，首先我们先看下任务栈的结构：  
[![2](http://b.hiphotos.bdimg.com/album/s%3D1400%3Bq%3D90/sign=c5ef28cb96eef01f49141cc1d0cea254/a08b87d6277f9e2f76688ca71d30e924b999f3c4.jpg)](http://b.hiphotos.bdimg.com/album/s%3D1400%3Bq%3D90/sign=c5ef28cb96eef01f49141cc1d0cea254/a08b87d6277f9e2f76688ca71d30e924b999f3c4.jpg)
　　当任第一次被调度的时候，我们也是需要恢复一些列的寄存器的，因为任务之前没有被调用过，所以此时在任务堆栈中并没有相关寄存器的状态，那么怎么实现任务的切换呢？MQX中已经帮我们做好了，当我们在创建任务调用\_psp\_build\_stack\_frame函数来创建任务堆栈时，设置了任务描述符的STACK\_BASE、STACK\_LIMIT和STACK\_PTR三个成员并且填充了其中的PSP\_STACK\_START\_STRUCT区域，代码如下：  
{% highlight c++ %}
boolean _psp_build_stack_frame
   (
      TD_STRUCT_PTR    td_ptr,
      pointer          stack_ptr,
      _mem_size        stack_size,
      TASK_TEMPLATE_STRUCT_PTR template_ptr,
      _mqx_uint        status_register,
      uint_32          create_parameter
   )
{
   uchar_ptr stack_base_ptr;
   PSP_STACK_START_STRUCT_PTR stack_start_ptr;
   boolean res = TRUE;

   stack_base_ptr  = (uchar_ptr)_GET_STACK_BASE(stack_ptr, stack_size);
   stack_start_ptr = (PSP_STACK_START_STRUCT_PTR)(stack_base_ptr - sizeof(PSP_STACK_START_STRUCT));

   td_ptr->STACK_BASE  = (pointer)stack_base_ptr;
   td_ptr->STACK_LIMIT = _GET_STACK_LIMIT(stack_ptr, stack_size);
   td_ptr->STACK_PTR   = stack_start_ptr;
   //初始化栈结构，用于第一次调度时任务的“返回”
   _mem_zero(stack_start_ptr, (_mem_size)sizeof(PSP_STACK_START_STRUCT));
   stack_start_ptr->INITIAL_CONTEXT.LR = (uint_32)_task_exit_function_internal;
   stack_start_ptr->INITIAL_CONTEXT.R0 = (uint_32)create_parameter;
   stack_start_ptr->INITIAL_CONTEXT.PC = (uint_32)(template_ptr->TASK_ADDRESS) | 1;
   //EPSR的T位，等于PC的位[0]，必须为1，表明thumb状态
   stack_start_ptr->INITIAL_CONTEXT.PSR = 0x01000000;
   //参数
   stack_start_ptr->PARAMETER = create_parameter;
   //PENDSVPRIOR = 0
   stack_start_ptr->INITIAL_CONTEXT.PENDSVPRIOR = 0;
   //TASK_SR-->BASEPRI
   stack_start_ptr->INITIAL_CONTEXT.BASEPRI     = status_register;
   //LR2 = 0xfffffffd，返回线程模式，使用PSP
   stack_start_ptr->INITIAL_CONTEXT.LR2         = 0xfffffffd;
   return res;
}
{% endhighlight %}
　 在任务第一次被调度的时候，由于之前内核并没有为其自动完成入栈（R0~R3、R12、LR、PC、xPSR，《MQX机制分析——中断机制(三)》中有讲，实际顺序在内部会被打乱，中断机制(三）中为真实顺序），所以通过这个函数，我们手工填充了自动堆栈部分，这样，当任务被调度的时候，就从我们手工设置的值进行恢复，我们再回到SVC和PendSV执行流程图中，当任务第一次被调度的时候，程序从任务堆栈的顶部将{r2-r11, lr}弹出堆栈，对应功能：

    r2 = PENDSVPRIOR;//PENDSV中断优先级为0
    r3 = BASEPRI;    //获取了任务的BASEPRI
    R4~R11 = R4~R11;
    LR = LR2(0xfffffffd);//将EXC_RETURN值赋给了LR寄存器，表明返回线程模式，使用PSP
　 再将此时的地址赋给PSP，此时PSP便指向了R0，接着使用了`bx lr`语句触发了中断返回序列（《MQX机制分析——中断机制(三)》中有讲），内核将R0~R3、R12、LR、PC、xPSR8个寄存器弹出（手工设置的值），这样之后PC=（template\_ptr->TASK\_ADDRESS) | 1，这里TASK\_ADDRESS是任务函数，R0是任务函数的参数create\_parameter（规定函数参数由R0~R3依次来保存，超过3个就使用堆栈，不过不需要用户来干预），返回地址在LR中，为\_task\_exit\_function\_internal（任务退出代码），这样就完成了从第1次启动调度时的任务切换的操作。  
　 如果是正常的任务切换的话，只是比上述过程多了将{r2-r11, lr}压入任务堆栈这一步，因为程序已经开始运行，运行的状态需要保存以便于恢复上下文使用。
