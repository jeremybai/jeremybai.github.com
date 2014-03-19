---
layout: post
title: "MQX机制分析——中断机制"
description: ""
categories: 
- C
tags: [MQX,操作系统,飞思卡尔]
---
{% include JB/setup %}

## MQX中断机制特点 ##
　　MQX的中断机制相对来说比较复杂，采用了随时注册随时使用的机制。MQX对中断处理采取如下措施对中断请求进行动态管理：
 
    　　（1）使用了静态中断向量表和稀疏向量表二张中断向量表；  
    　　（2）采用了内核ISR和用户ISR的分解处理机制；  
    　　（3）通过内核ISR把两张中断向量表联系在一起；  
    　　（4）采用了独立的系统中断栈。

　　在MQX中有两张完整的中断向量表：vectors\_rom（静态中断向量表）和vectors\_ram（动态中断向量表）。vectors\_rom在地址为0x00000000~0x00000400的地址里，而vectors\_ram情况比较特殊，如果宏定义`MQX_ROM_VECTORS`被定义了，那么编译的时候就不会产生这个段；如果没有定义，那它就在0x1FFF0000~0x1FFF0400的sram里。这两块地址里面存放的都是中断向量表，不过一个是固化在flash里的，一个是MQX运行之前加载到sram里的，唯一的区别就是前者的中断向量无法动态修改，而后者是可以随意改的。
{% highlight c++ %}
 __attribute__((section(".vectors_ram"))) vector_entry ram_vector[256] __attribute__((used)) =
 {
        (vector_entry)__BOOT_STACK_ADDRESS,
        BOOT_START,        /* 0x01  0x00000004   -  ivINT_Initial_Program_Counter*/
        _int_kernel_isr,   /* 0x02  0x00000008   -   ivINT_NMI                   */
        _int_kernel_isr,   /* 0x03  0x0000000C   -   ivINT_Hard_Fault            */
        _int_kernel_isr,   /* 0x04  0x00000010   -   ivINT_Mem_Manage_Fault      */
        _int_kernel_isr,   /* 0x05  0x00000014   -   ivINT_Bus_Fault             */
        _int_kernel_isr,   /* 0x06  0x00000018   -   ivINT_Usage_Fault           */
        0,                 /* 0x07  0x0000001C   -   ivINT_Reserved7             */
        0,                 /* 0x08  0x00000020   -   ivINT_Reserved8             */
        0,                 /* 0x09  0x00000024   -   ivINT_Reserved9             */
        0,                 /* 0x0A  0x00000028   -   ivINT_Reserved10            */
        _svc_handler,      /* 0x0B  0x0000002C   -   ivINT_SVCall                */
        _int_kernel_isr,   /* 0x0C  0x00000030   -   ivINT_DebugMonitor          */
        0,                 /* 0x0D  0x00000034   -   ivINT_Reserved13            */
        _pend_svc,         /* 0x0E  0x00000038   -   ivINT_PendableSrvReq        */
        _int_kernel_isr    /* 0x0F  0x0000003C   -   ivINT_SysTick               */
    };
{% endhighlight %} 
　　**如果使用的是动态中断向量表**（如下代码片段），那么会在编译的时候产生vectors\_ram段，将下面这张表放在0x1FFF0000~0x1FFF0400这一块地址，我们可以看到这张表只有16项，为什么它不像vectors\_rom那样将256项全放进去呢？因为这张表就是在ram中的，只要我们想要，随时都可以将后面所以的项都加进去，所以现在没有必要把它们全部放进去。在**创建完稀疏向量表之后**调用的\_psp\_int\_install函数中（如下代码片段），我们就可以看到程序将后面的项补全了。接着调用\_int\_set\_vector\_table函数来设置VTOR(Vector Table Offset Register，中断向量表偏移寄存器)寄存器来设置中断向量表的起始地址了。在有些情况情况下，一些实时的应用执行中断的延时可能小于MQX中段的延时，在这种情况下就可以使用\_int\_install\_kernel\_isr这个函数避开MQX从而使得中断直接响应。中断发生时，系统直接从动态中断向量表中调用用户自己定义的中断服务例程，实现对中断服务程序上半部的屏蔽。
{% highlight c++ %}
#if !MQX_ROM_VECTORS
    ptr = (uint_32_ptr)ram_vector;   //ram_vector就是动态中断向量表的起始地址
    for (i = 16; i < PSP_MAXIMUM_INTERRUPT_VECTORS; i++) {
        ptr[i] = (uint_32)_int_kernel_isr;
    }
#endif
{% endhighlight %} 
　　就算系统使用的是放在内存里的vectors_ram，但是不代表vectors_ram是毫无作用了。因为arm规定了cortex内核的芯片在上电复位的时候默认从地址为0x00000000的地方获取中断向量表，至于为什么，那是VTOR（Vector Table Offset Register）这个控制中断向量表位置的寄存器上电默认为0的缘故。vectors\_rom所存在的意义就是在给芯片上电到修改VTOR换到内存里的中断向量表vectors\_ram提供一个过渡。既然是个过渡的东西，所以除了最开始两个控制堆栈地址和复位地址以及一些特殊的异常向量之外其他都是DEFAULT\_VECTOR。
　　**MQX的源程序中因为定义了`MQX_ROM_VECTORS`，所以使用的是vectors_rom，也就是静态中断向量表**（如下代码片段）：    
{% highlight c++ %}
__attribute__((section(".vectors_rom"))) const vector_entry __vector_table[256] __attribute__((used)) = 
{
    (vector_entry)__BOOT_STACK_ADDRESS,
    BOOT_START,         /* 0x01  0x00000004   -   ivINT_Initial_Program_Counter */
    DEFAULT_VECTOR,     /* 0x02  0x00000008   -   ivINT_NMI                     */
    DEFAULT_VECTOR,     /* 0x03  0x0000000C   -   ivINT_Hard_Fault              */
    DEFAULT_VECTOR,     /* 0x04  0x00000010   -   ivINT_Mem_Manage_Fault        */
    DEFAULT_VECTOR,     /* 0x05  0x00000014   -   ivINT_Bus_Fault               */
    DEFAULT_VECTOR,     /* 0x06  0x00000018   -   ivINT_Usage_Fault             */
    0,                  /* 0x07  0x0000001C   -   ivINT_Reserved7               */
    0,                  /* 0x08  0x00000020   -   ivINT_Reserved8               */
    0,                  /* 0x09  0x00000024   -   ivINT_Reserved9               */
    0,                  /* 0x0A  0x00000028   -   ivINT_Reserved10              */
    _svc_handler,       /* 0x0B  0x0000002C   -   ivINT_SVCall                  */
    DEFAULT_VECTOR,     /* 0x0C  0x00000030   -   ivINT_DebugMonitor            */
    0,                  /* 0x0D  0x00000034   -   ivINT_Reserved13              */
    _pend_svc,          /* 0x0E  0x00000038   -   ivINT_PendableSrvReq          */
    DEFAULT_VECTOR,     /* 0x0F  0x0000003C   -   ivINT_SysTick                 */
    /* Cortex external interrupt vectors                                        */
    DEFAULT_VECTOR,     /* 0x10  0x00000040   -   ivINT_DMA0                    */
    DEFAULT_VECTOR,     /* 0x11  0x00000044   -   ivINT_DMA1                    */
    DEFAULT_VECTOR,     /* 0x12  0x00000048   -   ivINT_DMA2                    */
...
{% endhighlight %} 
　　因为MQX_ROM_VECTORS被定义了，所以中断向量表中那些DEFAULT_VECTOR都被定义为了_int_kernel_isr：  
{% highlight c++ %}
#if MQX_ROM_VECTORS
    #define DEFAULT_VECTOR  _int_kernel_isr
{% endhighlight %} 
　　这里有人可能就会产生疑惑，这张表是在rom区的，也就意味着这块地址是不可以被修改的，那么为什么会把这么多中断服务例程都定义为相同的\_int\_kernel\_isr函数呢？这里就要介绍到MQX中断处理机制的第二个特性：**采用了内核ISR和用户ISR的分解处理机制，也正是由于这一特性，使得MQX虽然使用的是静态中断向量表，却可以使得中断处理程序可以随时注册随时使用**。内核ISR和用户ISR的分解处理机制使得MQX对中断事件的快速响应，内核ISR用于实现硬件中断到用户ISR的映射，一般使用汇编语言实现；用户ISR通常由用户使用C语言编写，实现具体中断处理函数功能。这个概念与Linux的中断机制中的上半部下半部类似，可以对照理解。这里的\_int\_kernel\_isr这个函数就是使用汇编语言编写的，也就是我们所说的内核ISR，我们会在下面介绍它的具体功能，现在我们只需要知道的是在这个函数中，我们会根据中断号到稀疏中断向量表中去找用户ISR。这也就是第三个特性：通过内核ISR把两张中断向量表联系在一起。

　　第四个特性就是将系统中断栈与任务栈分开，提高了系统的安全性和稳定性。这一点与Linux类似，在2.6以前的Linux内核版本中，中断处理程序是没有自己的栈的，他们共享的是所中断进程的内核栈（大小两页，32位体系结构是8KB，64位体系是16KB），使用起来会比较节约。之后在2.6早期的内核中，增加了一个选项，把栈的大小从两页减为一页，减少了内存的压力，但是这样中断所用的栈更很少了，于是中断处理程序就拥有了自己的栈，尽管大小只有原来的一半，但是平均可以利用的栈大了很多，因为中断处理程序可以使用整整一页。MQX也是采用的这种思想所以使用了独立的中断栈。


## 稀疏中断向量表是什么 ##
　　什么我们提到了稀疏中断向量表，那么它到底是什么呢？首先我们看它在哪里创建的，首先\_mqx()对系统初始化中调用了\_bsp\_enable\_card()函数初始化芯片外设，其次在初始化芯片外设中调用了\_psp\_int\_init()中断初始化函数，再次在中断初始化函数中调用了\_int\_init( )函数，创建了稀疏中断向量表头节点并进行初始化。

　　稀疏中断向量表是按照HASH结构组织的一张表，如下图所示，稀疏中断向量表表项以8个向量为一组（8为缺省值分组值，可以修改）管理。系统初始化时根据给定的中断向量的数量除以8来确定稀疏中断向量表的组数，如251个中断可以分成251/8为32个组（进一法得到组数）。MQX系统初始化时调用了一个malloc函数为这32个成员申请一块连续的地址空间（数组成员为void*类型，每个指针为4字节），所以成员之间可以通过偏移一个队头节点大小（4字节）的空间进行寻址；而每个成员又是一个链表的头指针，链表各个表项节点地址不连续，每个链表项节点是独立的动态申请；通过链表项内的“NEXT”字段指针串成链表。MQX初始化时将每个组头节点的内容都初始设为NULL。链表节点内容由用户在安装中断服务例程时，调用\_int\_install\_isr函数填充。链表节点包含了中断向量号，对应的ISR的指针，传递给ISR的参数以及链表中下一个中断向量表表项元素的指针等等。
[![表1](http://a.hiphotos.bdimg.com/album/s%3D1400%3Bq%3D90/sign=be1fec7f3887e9504617f76820086832/d1160924ab18972bdc18e376e4cd7b899e510a6f.jpg)](http://a.hiphotos.bdimg.com/album/s%3D1400%3Bq%3D90/sign=be1fec7f3887e9504617f76820086832/d1160924ab18972bdc18e376e4cd7b899e510a6f.jpg)  

　　用户ISR由用户自定义编写，可以在运行时动态安装到MQX的中断处理系统中。如果用户需要实现一个ISR，只需要将该中断的中断号、该ISR的指针以及其他参数传递给\_int\_install\_isr函数就行了。\_int\_install\_isr函数将用户ISR与内核ISR绑定，它在稀疏中断向量表中根据参数中的中断号找到它所属的组（假如所要绑定中断的中断号为15，只需要计算（15-0）/8=1就可以得到它所在的组的地址为kernel\_data->INTERRUPT\_TABLE\_PTR[1]，也就是第二组），然后遍历整个链表查找与它有着相同中断号的节点， 若没有，就添加新的表项；若已经存在，就将之前的ISR覆盖。所以即使重复安装一个中断的用户ISR，也不会使某一组中的表项个数超过8个，因为中断安装函数做的覆盖之前旧的用户ISR。  

	
