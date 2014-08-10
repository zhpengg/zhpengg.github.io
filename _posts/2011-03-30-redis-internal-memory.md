---
layout: post
title: Redis Internal - Memory
description: 
category: "system"
tags: ['redis']
---

这是一篇翻译文章，原文地址在[这里](http://redis.io/topics/internals-vm)。 本文将对Redis内部的虚拟存储子系统做出说明。本文的目标读者并不是Redis的最终用户，而是那写想要了解并修改虚拟存储实现的程序员们。 

### 键vs.值：将谁交换出去?

加入虚拟存储子系统的主要目的是让Redis能够自由的将内存中存储的对象交换到磁盘上。这是一个普遍的想法，但是针对Redis特定的是，它只会交换和值相关联的对象。为了更好的理解这一观点，我们将会使用DEBUG命令展示：从Redis内部看，一个键-值对是如何组织的。 
    
    
    redis> set foo bar
    OK
    redis> debug object foo
    Key at:0x100101d00 refcount:1, value at:0x100101ce0 refcount:1 encoding:raw serializedlength:4
    

正如我们所见到的，Redis最上层的哈希表将一个Redis对象（这里是“键”）映射到另一个Redis对象上（“键”所对应的“值”）。虚拟存储仅仅能够将“值”交换到磁盘上，而所对应的“键”将会一直保存在内存中。这样的策略会带来非常好的查询性能，这也是Redis虚拟存储子系统设计时的一个主要目标：当最经常访问的数据集存放于内存中时，使用Redis虚拟存储子系统和关闭Redis虚拟存储子系统具有相似的性能。 

### 交换出去的“值”在内部是什么样的？

当一个对象被交换出去的时候，哈希表的表项将会发生如下变化： 

  * “键”依然保存着Redis对象来表示“键” 
  * “值”设为NULL 那么你可能会问，我们应该如何保存被交换出去的“值”（与一个给定的“键”对应的）的存放信息呢？答案是：在保存“值”的对象中。 这是Redis对象结构robj的定义： 
    
    
{% highlight c %}
    /* The actual Redis Object */
    typedef struct redisObject {
        void \*ptr;
        unsigned char type;
        unsigned char encoding;
        unsigned char storage;  /* If this object is a key, where is the value?
                                 * REDIS_VM_MEMORY, REDIS_VM_SWAPPED, ... */
        unsigned char vtype; /* If this object is a key, and value is swapped out,
                              * this is the type of the swapped out object. */
        int refcount;
        /* VM fields, this are only allocated if VM is active, otherwise the
         * object allocation function will just allocate
         * sizeof(redisObjct) minus sizeof(redisObjectVM), so using
         * Redis without VM active will not have any overhead. */
        struct redisObjectVM vm;
    } robj;
{% endhighlight %}

正如我们所见到的，结构中包含了一些和虚拟存储相关的字段。最重要的一个字段是storage字段，它可以是以下的一个值： 

  * REDIS_VM_MEMORY - “值”在内存中 
  * REDIS_VM_SWAPPED - “值”被交换出去了，并且哈希表中“值”的表项被设置成了NULL 
  * REDIS_VM_LOADING - “值”被交换到了磁盘中，且哈希表中“值”的表项为NULL，但是当前有一个任务正将对象从磁盘载入到内存（这个字段仅在线程化虚拟存储开启时才用到）。 
  * REDIS_VM_SWAPPING - “值”处于内存中，哈希表项中存放的是Redis对象的指针，但是当前有一个I/O任务正在将这个对象交换到磁盘上。  如果一个对象被交换到了磁盘上（RDEIS_VM_SWAPPED或是REDIS_VM_LOADING），我们怎样知道它的确切存储位置呢，以及它是什么类型的等等？很简单：vtype字段就是用来存放被交换出去的对象的原始类型的，同时使用vm字段（它是一个redisObjectVM类型的结构体）来保存有关对象的位置信息。下边的是这个结构体的定义： 
    
    
    struct redisObjectVM {
        off_t page;         //the page at which the object is stored on disk 
        off_t usedpages;    // number of pages used on disk 
        time_t atime;       // Last access time 
    } vm;
    

正如你所见到的，结构体中保存了对象被交换出去后做存放的交换文件的页码（page），所使用的页数以及最后一次访问时间（当要挑选一个即将载入内存的候选页时，这个字段是很有用的，因为我们想将最少访问的对象交换到磁盘中）。 正如你所见，在旧的Redis对象结构体内其他所有的字段都使用的是未使用的字节（？）vm字段是新的，确实需要更多的内存。当我们关闭虚拟存储的时候，我们还需要这些开销吗？显然不是的！下边的代码用来建立一个新的Redis对象： 
    
    
    ... some code ...
            if (server.vm_enabled) {
                pthread_mutex_unlock(&server.obj;_freelist_mutex);
                o = zmalloc(sizeof(*o));
            } else {
                o = zmalloc(sizeof(*o)-sizeof(struct redisObjectVM));
            }
    ... some code ...
    

如你所见，当虚拟存储没有启用的时候，我们仅仅分配了 `sizeof(*o)-sizeof(struct redisObjectVM)` 大小的内存。我们让vm字段作为对象结构体的最后一个字段，这样当虚拟存储关闭的时候，这个字段就永远不会被访问到。这样做是安全的，而且禁用虚拟存储的Redis不用花费无用的代价。 

### 交换文件

为了弄明白虚拟存储子系统是如何工作的，下一步就是察看对象是怎样存放在交换文件中的。好消息是这并不是什么特殊的格式，我们用保存.rdb文件一样的格式保存交换文件，这种文件是当我们使用SAVE命令转储时产生的文件。 交换文件是由给定数目的页构成的，页的大小则由给定数字确定，单位是字节。因为不同的Redis实例使用不同的参数可能获得更好的性能，所以这些参数都是可以从redis.conf文件中修改的：这取决于你实际存储的数据大小。下边的这个是默认值： 
    
    
    vm-page-size 32
    vm-pages 134217728
    

Redis在内存中保存了一张位图（一段连续的数组，每一位置0或1），其中的每一位代表了磁盘中交换文件的页：如果给定的位置1则表示这一页已经被使用了（也中存储了Redis对象数据），相反的是，当给定位置0，则表示对应的页是空闲的。 将这个位图（接下来我们称之为“页表”）保存在内存中将会极大的提升系统性能，同时对内存的使用也很少：我们只需内存中的一位就能表示磁盘中的一页。举例来说：134217728个页面，每一个页面大小是32字节（也就是一共4G的交换文件），整个页表仅占用内存16M。 

### 从内存中将对象交换到交换区

为了将对象从内存交换到磁盘中，我们按照如下步骤去做（假设采用非线程化的虚拟存储，仅采用简单的阻塞式做法）： 

  * 计算将对象存储到交换区需要多少页。通过调用rdbSavedObjectPages函数来计算，函数会返回对象存储磁盘中所需要的页数。需要注意的是，这个函数并没有复制保存.rdb文件的代码，仅仅是用来获得将对象保存到磁盘所需的空间大小。我们用了个小技巧，将对象写到/dev/null中，之后调用ftello获得所需的字节数。基本上我们做的就是将对象写入一个快速的虚拟文件，我们选择的是/dev/null。 
  * 现在我们知道了需要多少页，接下需要在交换文件中寻找连续的可用页。我们通过调用vmFindContiguousPages函数来完成。你可能已经猜到了，当交换区满了或是找不到连续可用空间的时候，函数会执行失败。出现这种情况的时候，我们仅仅是中断这个交换过程，对象仍将存储在内存中。 
  * 最后将对象写入到磁盘中，完成这一任务仅需调用vmWriteObjectOnSwap函数。 你可能已经猜到了，一旦对象成功写到了交换区中，那么对象之前所占用的内存就会释放了。同时与存储相关的键就会设置为REDIS_VM_SWAPPED，并且所占用的页也会在页表中标记。 

### 载入对象到内存

