---
layout: post
title: 使用GDB调试Coredump文件
description: 
category: "system"
tags: ['linux']
---

写C/C++程序经常要直接和内存打交道，一不小心就会造成程序执行时产生Segment Fault而挂掉。一般这种情况都是因为数组越界访问，空指针或是野指针读写造成的。程序小的话还比较好办，对着源代码仔细检查就能解决。但是对于代码量较大的程序，里边包含N多函数调用，N多数组指针访问，这时想定位问题就不是很容易了（此时牛人依然可以通过在适当位置打printf加二分查找的方式迅速定位:P）。懒人的话还是直接GDB搞起吧。

## 神马是Core Dump文件

偶尔就能听见某程序员同学抱怨“擦，又出Core了！”。简单来说，core dump说的是操作系统执行的一个动作，当某个进程因为一些原因意外终止（crash）的时候，操作系统会将这个进程当时的内存信息转储（dump）到磁盘上1。产生的文件就是core文件了，一般会以core.xxx形式命名。 

## 如何产生Core Dump

发生doredump一般都是在进程收到某个信号的时候，Linux上现在大概有60多个信号，可以使用 kill -l 命令全部列出来。 
    
    
{% highlight bash %} 
$ kill -l
  1) SIGHUP   2) SIGINT   3) SIGQUIT  4) SIGILL   5) SIGTRAP
  6) SIGABRT     7) SIGBUS   8) SIGFPE   9) SIGKILL 10) SIGUSR1
  11) SIGSEGV   12) SIGUSR2 13) SIGPIPE 14) SIGALRM 15) SIGTERM
  16) SIGSTKFLT 17) SIGCHLD 18) SIGCONT 19) SIGSTOP 20) SIGTSTP
  21) SIGTTIN   22) SIGTTOU 23) SIGURG  24) SIGXCPU 25) SIGXFSZ
  26) SIGVTALRM 27) SIGPROF 28) SIGWINCH    29) SIGIO   30) SIGPWR
  31) SIGSYS    34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
  38) SIGRTMIN+4    39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
  43) SIGRTMIN+9    44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
  48) SIGRTMIN+14   49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
  53) SIGRTMAX-11   54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
  58) SIGRTMAX-6    59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
  63) SIGRTMAX-1    64) SIGRTMAX    
{% endhighlight %} 

针对特定的信号，应用程序可以写对应的信号处理函数。如果不指定，则采取默认的处理方式, 默认处理是coredump的信号如下： 
        
{% highlight bash %}  
  3)SIGQUIT   4)SIGILL    6)SIGABRT   8)SIGFPE    11)SIGSEGV    7)SIGBUS    31)SIGSYS
  5)SIGTRAP   24)SIGXCPU  25)SIGXFSZ  29)SIGIOT   
{% endhighlight %}

我们看到SIGSEGV在其中，一般数组越界或是访问空指针都会产生这个信号。另外虽然默认是这样的，但是你也可以写自己的信号处理函数改变默认行为，更多信号相关可以看参考链接33。 

上述内容只是产生coredump的必要条件，而非充分条件。要产生core文件还依赖于程序运行的shell，可以通过ulimit -a命令查看，输出内容大致如下： 
    
{% highlight bash %}      
$ ulimit -a
  core file size          (blocks, -c) 0
  data seg size           (kbytes, -d) unlimited
  scheduling priority             (-e) 20
  file size               (blocks, -f) unlimited
  pending signals                 (-i) 16382
  max locked memory       (kbytes, -l) 64
  max memory size         (kbytes, -m) unlimited
  open files                      (-n) 1024
  pipe size            (512 bytes, -p) 8
  POSIX message queues     (bytes, -q) 819200
  real-time priority              (-r) 0
  stack size              (kbytes, -s) 8192
  cpu time               (seconds, -t) unlimited
  max user processes              (-u) unlimited
  virtual memory          (kbytes, -v) unlimited
  file locks                      (-x) unlimited
{% endhighlight %}

看到第一行了吧，core file size，这个值用来限制产生的core文件大小，超过这个值就不会保存了。我这里输出是0，也就是不会保存core文件，即使产生了，也保存不下来==! 要改变这个设置，可以使用ulimit -c unlimited。 

OK, 现在万事具备，只缺一个能产生Core的程序了，介个对C程序员来说太容易了。 
     
{% highlight c %} 
  #include <stdlib.h>;
  #include <stdio.h>;

  int crash()
  {
    char *xxx = "crash!!";
    xxx[1] = 'D'; // 写只读存储区!
    return 2;
  }

  int foo()
  {
    return crash();
  }

  int main()
  {
    return foo();
  }
{% endhighlight %}    

## 上手调试

上边的程序编译的时候有一点需要注意，需要带上参数-g, 这样生成的可执行程序中会带上足够的调试信息。编译运行之后你就应该能看见期待已久的“Segment Fault(core dumped)”或是“段错误 (核心已转储)”之类的字眼了。看看当前目录下是不是有个core或是core.xxx的文件。祭出linux下经典的调试器GDB，首先带着core文件载入程序：gdb exefile core，这里需要注意的这个core文件必须是exefile产生的，否则符号表会对不上。载入之后大概是这个样子的： 
    
{% highlight c %} 
  $ gdb coredump core
  Core was generated by ./coredump'.
      Program terminated with signal 11, Segmentation fault.
  #0  0x080483a7 in crash () at coredump.c:8
      8       xxx[1] = 'D';
  (gdb) 
{% endhighlight %}    

我们看到已经能直接定位到出core的地方了，在第8行写了一个只读的内存区域导致触发Segment Fault信号。在载入core的时候有个小技巧，如果你事先不知道这个core文件是由哪个程序产生的，你可以先随便找个代替一下，比如/usr/bin/w就是不错的选择。比如我们采用这种方法载入上边产生的core，gdb会有类似的输出： 
    
{% highlight bash %} 
$ gdb /usr/bin/w core
  Core was generated by ./coredump'.
  Program terminated with signal 11, Segmentation fault.
  #0  0x080483a7 in ?? ()
  (gdb) 
{% endhighlight %}    

可以看到GDB已经提示你了，这个core是由哪个程序产生的。 

## GDB 常用操作

上边的程序比较简单，不需要另外的操作就能直接找到问题所在。现实却不是这样的，常常需要进行单步跟踪，设置断点之类的操作才能顺利定位问题。下边列出了GDB一些常用的操作。 

  * 启动程序：run 
  * 设置断点：b 行号|函数名 
  * 删除断点：delete 断点编号 
  * 禁用断点：disable 断点编号 
  * 启用断点：enable 断点编号 
  * 单步跟踪：next 也可以简写 n 
  * 单步跟踪：step 也可以简写 s 
  * 打印变量：print 变量名字 
  * 设置变量：set var=value 
  * 查看变量类型：ptype var 
  * 顺序执行到结束：cont 
  * 顺序执行到某一行： util lineno 
  * 打印堆栈信息：bt 

bt这个命令重点推荐，尤其是当函数嵌套很深，调用关系复杂的时候，他能够显示出整个函数的调用堆栈，调用关系一目了然。另外上边有两个单步执行的命令，一个是n，一个是s。主要区别是n会将函数调用当成一步执行，而s会跟进调用函数内部。 

## 参考

  1. <http://en.wikipedia.org/wiki/Coredump>
  2. <http://www.kernel.org/doc/man-pages/online/pages/man5/core.5.html>
  3. <http://www.kernel.org/doc/man-pages/online/pages/man7/signal.7.html>
  4. <http://cs.baylor.edu/~donahoo/tools/gdb/tutorial.html>
  5. <http://bloggerdigest.blogspot.com/2006/09/gnu-gdb-core-dump-debugging.html>
