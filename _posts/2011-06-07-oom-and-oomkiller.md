---
layout: post
title: Oom and Oom-Killer
description: 
category: "system"
tags: ['linux']
---

Out of memory 也就是 OOM，指的是操作系统用光了所有内存（包括物理内存和交换区）时的情况。此时如果调用 malloc 函数申请内存的时候将会返回NULL，并且将 errno 设置为 ENOMEM。在引入虚拟内存和交换分区机制之后，OOM情形就很少出现了。但是即便如此，程序中还是应该谨慎的检查是否出现了OOM状况。  Linux系统中，当内核检测到OOM时会运行称作OOM-Killer的过程。以此来杀死一些经过精心挑选的进程，从而释放出他们占用的内存。oom-killer 的宗旨是牺牲一个或一组进程，来保证系统的正常运行。其实现在内核代码中的 mm/oom_kill.c 文件中。如果不想某个进程被 oom-killer 误杀，可以将 /proc//oomadj 设置为 OOM_DISABLE（实际值为 -17）。 
    
    
    #echo -17 > /proc//oomadj

当调用 malloc 因为oom而出错返回时，将会引发下面一些列调用： 
    
    
    _alloc_pages -> out_of_memory() -> select_bad_process() -> badness()
    

最终是通过 badness 函数找到要杀死的进程。这个函数有自己的一套算法来给所有的进程打分，让后将分数返回给 select_bad_process 函数。一个合适的进程应该满足如下条件： 

  1. 造成的损失尽可能小。 
  2. 能回收的内存尽可能的大。 
  3. 不要杀死无辜的进程。 
  4. 杀死的进程树要尽可能的少，1个最好。 
  5. 杀死那些用户最想杀死的进程。  对每个进程的打分从他所占用的内存开始： 
    
    
    /* The memory size of the process is the basis for the badness.*/
    points = p->mm->total_vm;
    

同时为了避免一个进程通过 fork 大量子进程来逃避检查，这个进程（内核线程除外）的所有子进程占用的内存也加到这个分数上： 
    
    
    if (chld->mm != p->mm && chld->mm)
                            points += chld->mm->total_vm;
    

Nice值高的进程一般都是交互性强的，所以相比长时间在后台运行的进程，杀掉nice值高的可能更安全一些。所以nice值高的进程，得到的分数更高。 
    
    
    s = int_sqrt(cpu_time);
    if (s)
        points /= s;
    s = int_sqrt(int_sqrt(run_time));
    if (s)
        points /= s;
    
    if (task_nice(p) > 0)
        points *= 2;
    

超级用户的进程要比普通用户的进程更加重要，所以要降低超级用户进程的分数。同时对于直接和硬件打交道的进程，最好还是不要强制杀死，因为这样可能会导致硬件出问题。 
    
    
    if (cap_t(p->cap_effective) & CAP_TO_MASK(CAP_SYS_ADMIN) ||
            p->uid == 0 || p->euid == 0)
    points /= 4;
    
    if (cap_t(p->cap_effective) & CAP_TO_MASK(CAP_SYS_RAWIO))
        points /= 4;
    

最终将这个进程的得分按照用户设定的/proc//oomadj 值进行移位，得出最终结果： 
    
    
    if (p->oomkilladj) {
        if (p->oomkilladj > 0)
            points <<= p->oomkilladj;
        else
            points >>= -(p->oomkilladj);
    }
    

### Reference

[Out of memory](http://en.wikipedia.org/wiki/Out_of_memory) [glibc内存管理ptmalloc2的实现](http://rdc.taobao.com/blog/cs/?p=1015) [OOM Killer](http://linux-mm.org/OOM_Killer)
