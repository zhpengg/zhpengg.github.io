---
layout: post
title: 线程库个种类与对比
description: 
category: "system"
tags: ['linux']
---

### Linux Thread

#### Linux 2.6 之前

在2.6版本之前的内核中，linux的线程库对POSIX线程仅提供部分支持，同时也存在一些问题。因为在早期的内核中，进程是最小的调度单位，所以线程库使用clone系统调用建立新的进程同时让新建立的进程共享父进程的地址空间来模拟线程。这样会在信号处理，线程调度，线程间同步产生一些问题。比如在进行信号处理的时候，因为每个线程的标示符实际是进程ID，所以各个线程没有一致的标示符，内核使用SIGUSR1和SIGUSR2两个信号进行线程间通信，与此同时用户就不能再使用这两个信号了。 

#### Native POSIX Thread Library(NPTL)

为了克服上述的不足，产生了两个相互竞争的工程：IBM领导的NGPT（Next Generation Posix Thread）和 Redhat 领导的 NPTL（Native POSIX Thread Library），最后 NPTL 胜出，加入到2.6版内核中。NPTL 的线程同样是使用clone() 系统调用产生的进程，区别是，NPTL 要求内核提供原生的根据条件唤醒和休眠线程的操作，以及一些其他特性。NPTL 使用的是1对1的线程模型，也就是说每当用户调用pthread_create() 产生一个线程，内核中就对应这一个可调度的实体。 

### The GNU Portable Thread(GNU Pth)

Pth同样是适用于Unix平台的，兼容POSIX标准的一个线程库。它的最大优点是基于优先级的非抢占式调度算法。线程的调度是通过一个给予优先级和事件的调度器实现的。设计者认为这样的调度策略能提供相对抢占式调度更好的运行时性能。 

### Windows Thread

相比 linux 中使用轻量级进程实现线程，Windows对线程的实现要比其对进程的实现要轻量级的多。具体参考引用中链接。另外Redhat 试图将POSIX线程库移植到Win32平台，貌似已经停止更新了。 

### Open Multi-Processing (Open MP)

Open MP 由一些主流的硬件厂商和软件厂商提出的，目标是提供一个跨平台（同时支持 Unix 和 Windows 平台），支持多语言（C，C++，Fortran等）的应用编程接口。同时还支持集群的并行编程。 

### Reference

[Native Posix thread library](http://en.wikipedia.org/wiki/Native_POSIX_Thread_Library) [The GNU Portable Thread](http://www.gnu.org/software/pth/) [Open Muti-Processing](http://en.wikipedia.org/wiki/OpenMP) [Windows Process And Thread](http://msdn.microsoft.com/en-us/library/ms681917%28v=vs.85%29.aspx) [Open Source POSIX for Win32](http://sources.redhat.com/pthreads-win32/) [Open MP](http://en.wikipedia.org/wiki/OpenMP)
