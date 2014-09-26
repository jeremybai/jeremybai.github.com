---
layout: post
title: "MQX机制分析——动态内存管理"
description: ""
categories: 
- C
tags: [MQX,操作系统,飞思卡尔]
---
{% include JB/setup %}

![1](http://github-blog.qiniudn.com/2014-03-11-mqx-memory-manage-1.jpg-BlogPic)
### 1 默认内存管理函数的不足 ###
　　我们知道c/c++普通的内存管理需要我们可以通过malloc/free和new/delete实现，在利用默认的这些内存管理函数在堆上分配和释放内存时会有一些额外的开销。系统在接收到分配一定大小内存的请求时，首先查找内部维护的内存空闲块表，并且需要根据一定的算法（例如分配最先找到的不小于申请大小的内存块给请求者，或者分配最适于申请大小的内存块，或者分配最大空闲的内存块等）找到合适大小的空闲内存块。如果该空闲内存块过大，还需要切割成已分配的部分和较小的空闲块。然后系统更新内存空闲块表，完成一次内存分配。类似地，在释放内存时，系统把释放的内存块重新加入到空闲内存块表中。如果有可能的话，可以把相邻的空闲块合并成较大的空闲块。默认的内存管理函数还考虑到多线程的应用，需要在每次分配和释放内存时加锁，同样增加了开销。可见，如果应用程序频繁地在堆上分配和释放内存，则会导致性能的损失。并且会使系统中出现大量的内存碎片，降低内存的利用率。默认的分配和释放内存算法自然也考虑了性能，然而这些内存管理算法的通用版本为了应付更复杂、更广泛的情况，需要做更多的额外工作[1]。  
### 2 MQX内存管理介绍 ###
　　而对于特定的RTOS和应用来说，适合自身特定的内存分配释放模式的自定义内存池则可以获得更好的性能。在MQX中线程即任务，任务至上的理念使得MQX的动态内存管理也与任务密不可分。通过MQX的API申请的内存根据内存资源的从属关系可将内存划分为属于任务的私有内存块和系统应用的系统内存块。任务的私有内存块属于该任务本身，只有该任务才可以释放这一块内存；系统内存属于系统任务--system_task，不知道大家还记不记得，在mqx启动的时候创建的这个系统任务，这个任务既没有对应的函数体，也没有被任务代码所启动，难道只是个摆设吗？存在既是合理的，后来才发现虽然这个任务不会被执行，但是它是为了迎合MQX中任务至上的观念，将一些不适合放在其他任务中的数据结构放在system_task的任务描述符中，比如说在初始化时创建的内存池，其所属的任务就是system_task[2]。系统内存块可以被任何的任务释放。那么是不是说一块内存只能被申请它的任务所释放呢？MQX中为我们提供了一个后门，我们可以通过\_lwmem_transfer/\_mem_transfer函数来将一块内存的所有权转移到另外一个任务上，间接的达到让另外一个任务释放该内存的目的。   
　　由任务来管理动态内存块还有一个好处，就是我们并不用太担心内存块的释放问题了。而对于MQX而言，一旦一个删除一个任务，就会将这个任务所拥有的全部动态内存全部删除，在_task_destroy_internal中会判断如果任务是不是自己删除自己，如果是的话直接释放内存，不是的话则需要获取当前任务TASK_NUMBER（TASK_ID的后16位，前16为是PROCESSOR_ID），将需要删除的内存块的所有权转让到当前任务的名下，然后再进行删除。  
　　MQX的内存管理如果从内存的分配大小上分的话有可变大小内存管理和固定大小内存管理。可变大小内存管理是根据任务申请多大就会分配多大，而固定大小内存管理中的内存被划分成一小块一小块，每次任务申请多少，都会按照这个最小块的整数倍数进行分配。可变大小的内存管理使用起来更加灵活，但是容易产生碎片，固定大小的内存管理在使用效率上可能会稍逊色一点，但是不容易产生碎片。  
　　可变大小的内存块管理方式是系统默认的方式，MQX中可变大小的内存块管理方式分为非轻量级内存（Memory）管理和轻量级内存（LwMemory）管理两种。包含对内存池的创建、内存块的分配、测试、校验检测、错误判断、内存块使用权转让等功能。固定大小区块的内存管理方式是个可选性组件，是可变大小区块管理方式的一个补充，它适用于对内存资源分配操作频繁的情况。MQX针对内存动态分配算法进行了简化，通过整块分配整块回收的机制有效地减少了内存碎片的产生，节约了内存资源，提高了申请和释放一个区块的速度，缩短对内存管理的时间。但使用的灵活性降低[3]。  
　　下面主要以轻量级内存管理为例介绍下MQX中的动态内存管理是如何实现的。
### 3 轻量级内存管理的实现 ###
#### 3.1 初始化 ####
　　在之前讲到的MQX的启动过程中调用\_mem_init_internal来初始化内存池的。MQX默认使用的是轻量级内存管理（由MQX_USE_LWMEM_ALLOCATOR宏定义在config文件中配置），\_mem_init_internal被映射为\_lwmem_init_internal函数，该函数部分代码如下。
{% highlight c++ %}
...
    start = (void *) ((unsigned char *) kernel_data + sizeof(KERNEL_DATA_STRUCT));
    lwmem_pool_ptr = (LWMEM_POOL_STRUCT_PTR) start;
    kernel_data->KERNEL_LWMEM_POOL = (void *) lwmem_pool_ptr;
    start = (void *) ((unsigned char *) start + sizeof(LWMEM_POOL_STRUCT));
    _lwmem_create_pool(lwmem_pool_ptr, start, (unsigned char *) kernel_data->INIT.END_OF_KERNEL_MEMORY - (unsigned char *) start);
...
{% endhighlight %} 
　　该函数将ram中KERNEL_DATA末尾一直到\_KERNEL_DATA_END的地址作为整个内存池的大小，kernel_data->INIT.END_OF_KERNEL_MEMORY是在初始化的时候由初始化结构体进行赋值，END_OF_KERNEL_MEMORY来自于链接文件中指定的地址。   
[![2](http://github-blog.qiniudn.com/2014-03-11-mqx-memory-manage-2.png-BlogPic) ](http://github-blog.qiniudn.com/2014-03-11-mqx-memory-manage-2.png) 
　　MQX中的可变大小的内存是以block为单位的，每一块内存（空闲或者使用）都会被加上LWMEM_BLOCK_STRUCT头，结构如下，这个头证明了这块内存的大小以及所属的内存池，其中的成员U是一个共用体，若该块内存为已使用的内存块，则该成员中的TASK_NUMBER和MEM_TYPE指明了该内存所属的任务的TASK_NUMBER以及内存类型（任务内存块还是系统内存块）。
{% highlight c++ %}
typedef struct lwmem_block_struct
{
   /*! \brief The size of the block. */
   _mem_size      BLOCKSIZE;
   /*! \brief The pool the block came from. */
   _lwmem_pool_id POOL;
   /*!
    * \brief For an allocated block, this is the task ID of the owning task.
    * When on the free list, this points to the next block on the free list.
    */
   union {
      void       *NEXTBLOCK;
      struct {
         _task_number    TASK_NUMBER;
         _mem_type       MEM_TYPE;
      } S;
   } U;
} LWMEM_BLOCK_STRUCT, * LWMEM_BLOCK_STRUCT_PTR;
{% endhighlight %} 
　　初始化完成之后，图中mem_pool之后一直到\_KERNEL_DATA_END都被作为一整块空闲的内存块。
#### 3.2 申请空间 ####
　　\_lwmem_alloc和\_lwmem_alloc_system都是用来申请内存块的，区别是前者是由当前任务申请，后者是由系统任务申请。申请内存时我们需要知道需要内存的大小，该内存属于哪个任务，哪个内存池。所以在其内部调用\_lwmem_alloc_internal时将这些所需要的元素传递进去，此时MQX会从POOL_FREE_LIST_PTR开始寻找合适大小的内存块，合适大小指的就是找到的第一块内存大小大于等于需求内存的内存块。采用这种方案虽然快，但是这会导致系统在后面不能分配出大块的内存供其他任务使用。如果当前找到的内存块大于所需要的内存大小时，就将该块内存分为2部分，一部分供申请的空间的任务使用，被标记为已使用内存块，剩下的还是作为空闲内存块链接到原来的POOL_FREE_LIST_PTR对应的链表中，其中需要注意的是如果找到的空闲块是在POOL_FREE_LIST_PTR前面，那么需要重新定义POOL_FREE_LIST_PTR。还有个值得注意的地方在于在查找空闲块的循环之中，每次都会有开关中断的操作，如下所示：
{% highlight c++ %}
	/* Provide window for higher priority tasks */
	mem_pool_ptr->POOL_ALLOC_PTR = block_ptr;
	_int_enable();
	_int_disable();
	block_ptr = mem_pool_ptr->POOL_ALLOC_PTR;
{% endhighlight %} 
　　这可能是因为遍历每一块空闲内存块这个时间比较长，对于RTOS来说，这么长时间不开中断是不能接受的，于是在每次找一个空闲块之后就开下中断看看是否有高优先级的任务需要切换，使用POOL_ALLOC_PTR保存当前查找的位置，切换回来之后将POOL_ALLOC_PTR里面的值重新加载回去继续执行，这里用POOL_ALLOC_PTR来保存block_ptr的值是为了防止高优先级的任务也需要申请空间，被切换出去之后，高优先级任务有可能将block_ptr所指的空闲内存使用，此时切换回来block_ptr指向的可能已经不是空闲内存块了，而POOL_ALLOC_PTR是一个内存池所共享的，高优先级寻找空闲内存块的同时也需要保存将block_ptr保存到POOL_ALLOC_PTR中，执行完高优先级任务之后，POOL_ALLOC_PTR被赋值为POOL_FREE_LIST_PTR。这样再切换为低优先级任务的时候，恢复到block_ptr，继续从空闲内存块链表头开始查找。
#### 3.3 释放空间 ####
　　释放内存的过程与申请有点类似，但是更加复杂一点。\_lwmem_free和释放\_mem_free类似，在释放时，首先需要验证当前任务是否具有释放指定内存的资格，也就是查看该内存是否属于当前任务，不是的话再判断该内存是否是系统内存，都不是的话就返回错误。之后根据该内存块的地址来决定插入POOL_FREE_LIST_PTR的哪个位置。使用free_list_ptr变量来遍历空闲内存链表，首先判断是否在POOL_FREE_LIST_PTR前面，在POOL_FREE_LIST_PTR前面还有两种情况，地址相邻还是不相邻，相邻的话就更新POOL_FREE_LIST_PTR指针，同时将两块内存合并，不相邻的话就更新POOL_FREE_LIST_PTR指针，将原来的POOL_FREE_LIST_PTR连接到需要释放的内存之后。  
　　如果需要释放的内存块地址在POOL_FREE_LIST_PTR之后，那就要开始查找需要插入的位置，当需要释放的地址大于free_list_ptr并且小于free_list_ptr的下一块地址时，就找到了需要插入的位置，在插入之前还需要做个判断就是该块前后是否有相邻的空闲内存块，按照`先后再前`的原则进行判断是否需要合并。如果不需要则只需要将该块插入找到的空闲块free_list_ptr之后即可。在遍历POOL_FREE_LIST_PTR链表时与申请空间类似，也有开关中断的步骤，原理类似，只不过用POOL_FREE_PTR来是实现多任务的保护。
{% highlight c++ %}
    /* Provide window for higher priority tasks */
    mem_pool_ptr->POOL_FREE_PTR = free_list_ptr;
    _int_enable();
    _int_disable();
    free_list_ptr = mem_pool_ptr->POOL_FREE_PTR;
{% endhighlight %}
### 4 结论 ###
　　这里只是列举了轻量级内存管理进行分析，非轻量级的与之类似，可以申请的空间更大，固定大小的内存管理没做分析，觉得应该比这个更简单一些。MQX的内存管理相比于其他操作系统应该算是比较简单了，尤其是在查找可用内存块的时候找到可用就用，而不考虑是否合适，长时间运行可能会导致内存碎片比较多，大块内存申请可能会出现问题，不过内存分配速度和效率本身就是矛盾的，只能从是否合适考虑而不能从单一指标进行考虑。
## 参考文献 ##
[1] 《C++应用程序性能优化》6.1 自定义内存池性能优化的原理  
[2] [MQX心得之任务的动态内存管理](http://blog.chinaunix.net/uid-25788300-id-3803067.html)  
[3] 《嵌入式实时操作系统MQX应用开发技术》  