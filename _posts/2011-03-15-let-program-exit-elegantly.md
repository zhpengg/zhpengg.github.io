---
layout: post
title: "让你的程序优雅的结束"
description: 
category: "system"
tags: ["ramhost"]
---

写C程序最爽的是什么？我觉得是其中灵活的指针用法能让coder发挥各种想象力和创造力。那最郁闷的是啥？差不多就是N多指针满天飞之后程序一运行就直接segment fault了。这种运行时错误不像编译时错误有明显的错误提示，所以往往很难定位。今天看[redis](http://redis.io/)源代码，看见了一个比较不错的追踪此类问题的方式。 

### Segment fault是怎样产生的

一般导致segment fault错误的原因都是程序中非法访问了某块内存。当操作系统的内存保护机制发现某个进程访问了非法内存的时候会向此进程发送一个SIGSEGV信号，而如果此进程中没有相应的信号处理函数的话，就会执行默认的动作，一般都是直接杀死进程。这样进程就会在shell中提示一个segment fault并退出。 此时如果察看系统日志的话，将会发现类似的内容： 
    
{% highlight bash %}
    kernel: [21147.410307] a.out[9236]: segfault at ffffffff ip 0804839c sp bfa88cd8 
    error 6 in a.out[8048000+1000]
{% endhighlight %}

但是仅凭借这点提示还是很难准确定位到出错的位置，尤其是当整个程序非常大，函数嵌套很多的时候。这个时候就想起java的好了，人家catch exception之后能直接printStackTrace来显示异常的类型和发生位置。其实linux中也有这种机制。 

### 使用Backtrace跟踪进程信息

在redis源码(debug.c)中有一个自己实现的类似assert()的函数叫做_redisAssert(),如下： 
    
{% highlight c %}    
    void _redisAssert(char *estr, char *file, int line) {
        redisLog(REDIS_WARNING,"=== ASSERTION FAILED ===");
        redisLog(REDIS_WARNING,"==> %s:%d '%s' is not true",file,line,estr);
        #ifdef HAVE_BACKTRACE
        redisLog(REDIS_WARNING,"(forcing SIGSEGV in order to print the stack trace)");
        *((char*)-1) = 'x';
        #endif
    }
{% endhighlight %}
    

函数的最后一行有*((char*)-1) = 'x'，这里故意让程序访问了一个非法的地址空间，从而导致SEGSEGV信号产生。在redis.c中有针对SIGSEGV信号的处理函数：segvHandler()，去掉和这里讨论无关的内容之后如下： 
    
{% highlight c %}
    void segvHandler(int sig, siginfo_t *info, void *secret) {
        void *trace[100];
        char **messages = NULL;
        int i, trace_size = 0;
        ucontext_t *uc = (ucontext_t*) secret;
        struct sigaction act;
    
        trace_size = backtrace(trace, 100);
       
        messages = backtrace_symbols(trace, trace_size);
    
        for (i=1; i<trace_size; ++i)
            redisLog(REDIS_WARNING,"%s", messages[i]);
       /* free(messages); Don't call free() with possibly corrupted memory. */
    
        /* Make sure we exit with the right signal at the end. So for instance
         * the core will be dumped if enabled. */
        sigemptyset (&act.sa;_mask);
        /* When the SA_SIGINFO flag is set in sa_flags then sa_sigaction
         * is used. Otherwise, sa_handler is used */
        act.sa_flags = SA_NODEFER | SA_ONSTACK | SA_RESETHAND;
        act.sa_handler = SIG_DFL;
        sigaction (sig, &act;, NULL);
        kill(getpid(),sig);
    }
{% endhighlight %}
   

这个信号处理函数的关键在于调用backtrace()和backtrace_symbols()函数将进程正在运行状态中的堆栈信息打印到日志中，最后再将进程结束。首先看一下这两个函数的原型： 
    
{% highlight c %}    
    #include 
    
           int backtrace(void **buffer, int size);
    
           char **backtrace_symbols(void *const *buffer, int size);
    
{% endhighlight %}

首先backtrace函数，这如其函数名字一样，返回当前进程的栈帧信息。我们知道在程序中的函数调用主要是靠栈来实现的，当发生函数调用的时候，当前函数的一些临时变量和返回信息都会记录在栈中。当进程因为segment fault中断执行的时候，我们就可以通过backtrace函数打印出当前栈中的内容，进而就能确定到底非法的内存访问是产生在哪个函数中。 backtrace函数会将栈中的信息保存到buffer所指向的数组中，size指明buffer允许的最大长度。返回值为实际存放在buffer中的条目个数。这里有一点要注意的是，如果你想看到栈中的全部内容，就必须保证buffer数组足够的长。一般来说100就足够了。 backtrace有个缺点，就是它返回的buffer中保存的是栈帧的地址，直接这样打印出来的话，人眼很难察看。所以就有了backtrace_symbols函数。它的作用就是将buffer中的地址转换为可读的字符串。并将转换完的字符串保存在message所指向的内存中。这也就是说backtrace_symbols会在函数内部调用malloc申请一块内存来保存转换完的信息，所以如有必要，请一定记得free掉。 stackoverflow上有一段示例程序，如下： 
    
{% highlight c %}    
    #ifndef _GNU_SOURCE
    #define _GNU_SOURCE
    #endif
    #ifndef __USE_GNU
    #define __USE_GNU
    #endif
    
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    
    /* This structure mirrors the one found in /usr/include/asm/ucontext.h */
    typedef struct _sig_ucontext {
     unsigned long     uc_flags;
     struct ucontext   *uc_link;
     stack_t           uc_stack;
     struct sigcontext uc_mcontext;
     sigset_t          uc_sigmask;
    } sig_ucontext_t;
    
    void crit_err_hdlr(int sig_num, siginfo_t * info, void * ucontext)
    {
     void *             array[50];
     void *             caller_address;
     char **            messages;
     int                size, i;
     sig_ucontext_t *   uc;
    
     uc = (sig_ucontext_t *)ucontext;
    
     /* Get the address at the time the signal was raised from the EIP (x86) */
     caller_address = (void *) uc->uc_mcontext.eip;   
    
     fprintf(stderr, "signal %d (%s), address is %p from %p
    ", 
      sig_num, strsignal(sig_num), info->si_addr, 
      (void *)caller_address);
    
     size = backtrace(array, 50);
    
     /* overwrite sigaction with caller's address */
     array[1] = caller_address;
    
     messages = backtrace_symbols(array, size);
    
     /* skip first stack frame (points here) */
     for (i = 1; i < size && messages != NULL; ++i)
     {
      fprintf(stderr, "[bt]: (%d) %s
    ", i, messages[i]);
     }
    
     free(messages);
    
     exit(EXIT_FAILURE);
    }
    
    int crash()
    {
     char * p = NULL;
     *p = 0;
     return 0;
    }
    
    int foo4()
    {
     crash();
     return 0;
    }
    
    int foo3()
    {
     foo4();
     return 0;
    }
    
    int foo2()
    {
     foo3();
     return 0;
    }
    
    int foo1()
    {
     foo2();
     return 0;
    }
    
    int main(int argc, char ** argv)
    {
     struct sigaction sigact;
    
     sigact.sa_sigaction = crit_err_hdlr;
     sigact.sa_flags = SA_RESTART | SA_SIGINFO;
    
     if (sigaction(SIGSEGV, &sigact;, (struct sigaction *)NULL) != 0)
     {
      fprintf(stderr, "error setting signal handler for %d (%s)
    ",
        SIGSEGV, strsignal(SIGSEGV));
    
      exit(EXIT_FAILURE);
     }
    
     foo1();
    
     exit(EXIT_SUCCESS);
    }
{% endhighlight %}    

作者对SIGSEGV信号处理函数做了一点改善，为了能够打印出当前正在运行的函数，程序中保存了eip寄存器的内容。 另外值得注意的一点是当时用gcc的链接器编译程序的时候，加入-rdynamic 链接选项。也可以直接在gcc编译的时候加上-rdynamic选项，这样backtrace才会显示出函数名。上边程序的输出结果大致是这个样子： 
    
{% highlight c %}    
    signal 11 (Segmentation fault), address is (nil) from 0x8c50
    [bt]: (1) ./test(crash+0x24) [0x8c50]
    [bt]: (2) ./test(foo4+0x10) [0x8c70]
    [bt]: (3) ./test(foo3+0x10) [0x8c8c]
    [bt]: (4) ./test(foo2+0x10) [0x8ca8]
    [bt]: (5) ./test(foo1+0x10) [0x8cc4]
    [bt]: (6) ./test(main+0x74) [0x8d44]
    [bt]: (7) /lib/libc.so.6(__libc_start_main+0xa8) [0x40032e44]
{% endhighlight %}

这样我们通过backtrace就能在程序因为内存访问的原因崩溃的时候顺利找到出错的原因。 

### 参考

[Stack Backtracing Inside Your Program](http://www.linuxjournal.com/article/6391) [how-to-generate-a-stacktrace-when-my-gcc-c-app-crashes](http://stackoverflow.com/questions/77005/how-to-generate-a-stacktrace-when-my-gcc-c-app-crashes) [man backtrace](http://www.kernel.org/doc/man-pages/online/pages/man3/backtrace.3.html)


