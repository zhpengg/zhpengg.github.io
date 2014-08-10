---
layout: post
title: Redis internals - 内存分配
description: 
category: "system"
tags: ['redis', 'linux']
---

Redis中到处都会进行内存分配操作。为了屏蔽不同平台之间的差异，以及统计内存占用量等，Redis对内存分配函数进行了一层封装，程序中统一使用zmalloc，zfree一系列函数，位于zmalloc.h，zmalloc.c文中。  上边说过，封装就是为了屏蔽底层平台的差异，同时方便自己实现相关的统计函数。具体来说就是： 

  * 若系统中存在Google的TC_MALLOC库，则使用tc_malloc一族函数代替原本的malloc一族函数。 
  * 若当前系统是Mac系统，则使用 `<malloc/malloc.h>` 中的内存分配函数。 
  * 其他情况，在每一段分配好的空间前头，同时多分配一个定长的字段，用来记录分配的空间大小。源代码分别在 config.h 和 zmalloc.c 中： 
    
    
    #if defined(USE_TCMALLOC)
    #include 
    #if TC_VERSION_MAJOR >= 1 && TC_VERSION_MINOR >= 6
    #define HAVE_MALLOC_SIZE 1
    #define redis_malloc_size(p) tc_malloc_size(p)
    #endif 
    #elif defined(__APPLE__)
    #include 
    #define HAVE_MALLOC_SIZE 1
    #define redis_malloc_size(p) malloc_size(p)
    #endif 
    
    
    
    #ifdef HAVE_MALLOC_SIZE
    #define PREFIX_SIZE (0)       
    #else  
    #if defined(__sun)
    #define PREFIX_SIZE (sizeof(long long))
    #else                         
    #define PREFIX_SIZE (sizeof(size_t))
    #endif
    #endif
    

因为 tc_malloc 和 Mac平台下的 malloc 函数族提供了计算已分配空间大小的函数（分别是tc_malloc_size和malloc_size），所以就不需要单独分配一段空间记录大小了。而针对linux和sun平台则要记录分配空间大小。对于linux，使用sizeof(size_t)定长字段记录；对于sun os，使用sizeof(long long)定长字段记录。也就是上边源码中的 PREFIX_SIZE 宏。 那么这个记录有什么用呢？答案是，为了统计当前进程到底占用了多少内存。在 zmalloc.c 中，有这样一个静态变量： 
    
    
    static size_t used_memory = 0;
    

它记录了进程当前占用的内存总数。每当要分配内存或是释放内存的时候，都要更新这个变量。因为分配内存的时候，可以明确知道要分配多少内存。但是释放内存的时候，（对于未提供malloc_size函数的平台）仅通过指向要释放内存的指针是不能知道释放的空间到底有多大的。这个时候，上边提到的PREFIX_SIZE定长字段就起作用了，可以通过其中记录的内容得到空间的大小。zmalloc函数如下（去掉无关代码）： 
    
    
    void *zmalloc(size_t size) {
        void *ptr = malloc(size+PREFIX_SIZE);
    
        if (!ptr) zmalloc_oom(size);
        *((size_t*)ptr) = size;
        update_zmalloc_stat_alloc(size+PREFIX_SIZE,size);
        return (char*)ptr+PREFIX_SIZE;
    #endif
    }
    

看到在分配空间的时候，空间大小是size+PREFIX_SIZE。对于mac系统或是使用tc_malloc的情况，PREFIX_SIZE 为0。之后将ptr指针指向的空间前size_t中记录分配空间的大小。最后返回的是越过记录区的指针。zfree函数类似（去掉无关代码）： 
    
    
    void zfree(void *ptr) {
        void *realptr;
        size_t oldsize;
    
        if (ptr == NULL) return;
        realptr = (char*)ptr-PREFIX_SIZE;
        oldsize = *((size_t*)realptr);
        update_zmalloc_stat_free(oldsize+PREFIX_SIZE);
        free(realptr);
    #endif
    }
    

先将指针向前移动PREFIX_SIZE，然后取出分配空间时保存的空间长度。最后free整个空间。 update_zmalloc_stat_alloc(__n,__size) 和 update_zmalloc_stat_free(__n) 这两个宏负责在分配内存或是释放内存的时候更新used_memory变量。定义成宏主要是出于效率上的考虑。将其还原为函数，就是下边这个样子： 
    
    
    void update_zmalloc_stat_alloc(__n,__size)
    {
        do {
            size_t _n = (__n);
            size_t _stat_slot = (__size < ZMALLOC_MAX_ALLOC_STAT) ? __size : ZMALLOC_MAX_ALLOC_STAT;
            if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1));
            if (zmalloc_thread_safe) {
                pthread_mutex_lock(&used;_memory_mutex);
                used_memory += _n;
                zmalloc_allocations[_stat_slot]++;
                pthread_mutex_unlock(&used;_memory_mutex);
            } else {  
                used_memory += _n;  
                zmalloc_allocations[_stat_slot]++;  
            }  
        } while(0)
    }
    
    void update_zmalloc_stat_free(__n)
    {
        do {  
            size_t _n = (__n);  
            if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1));
            if (zmalloc_thread_safe) {  
                pthread_mutex_lock(&used;_memory_mutex);
                used_memory -= _n;
                pthread_mutex_unlock(&used;_memory_mutex);
            } else {
                used_memory -= _n;
            }  
        } while(0)
    }
    

代码中除了更新used_memory变量外，还有几个要关注的地方： 

  1. 先对_n的低位向上取整，最后_n变为sizeof(long)的倍数，比如对于32位系统，sizeof(long) == 100(二进制)，_n向上取整之后，低两位都变为0。 
  2. 如果进程中有多个线程存在，则在更新变量的时候要加锁。 
  3. 在zmalloc函数中还有一个统计量要更新：`zmalloc_allocations[]`。  在 zmalloc.c 中，zmalloc_allocations是这样定义的： 
    
    
    size_t zmalloc_allocations[ZMALLOC_MAX_ALLOC_STAT+1];
    

其作用是统计程序分配内存时，对不同大小空间的请求次数。统计的空间范围从1字节到256字节，大于256字节的算为256。统计结果通过调用 zmalloc_allocations_for_size 函数返回： 
    
    
    size_t zmalloc_allocations_for_size(size_t size) {
        if (size > ZMALLOC_MAX_ALLOC_STAT) return 0;
        return zmalloc_allocations[size];
    }
    

另一个对内存使用量的统计通过调用 zmalloc_used_memory 函数返回： 
    
    
    size_t zmalloc_used_memory(void) {
        size_t um;
    
        if (zmalloc_thread_safe) pthread_mutex_lock(&used;_memory_mutex);
        um = used_memory;
        if (zmalloc_thread_safe) pthread_mutex_unlock(&used;_memory_mutex);
        return um;
    }
    

另外 zmalloc.c 中，还针对不同的系统实现了 zmalloc_get_rss 函数，在linux系统中是通过读取/proc/$pid/stat文件获得系统统计的内存占用量。关于/proc虚拟文件系统可以看[之前的文章](/?p=70)。 \--EOF

## Comments

**[ydzhang](#680 "2011-08-30 17:23:16"):** 统计除了内存使用情况，及内存块的分配情况，这些信息用于做什么呢？ 那些地方根据这些信息做了优化？另外为什么统计1-256字节这么细粒度的分配信息，大部分都是小内存分配么？

