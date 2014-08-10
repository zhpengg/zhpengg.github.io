---
layout: post
title: 即将消逝的目录 /var/run
description: 
category: "system"
tags: ['linux']
---

首先声明这不是愚人节消息，事实上[这个消息](http://lwn.net/Articles/436012/)是昨天就发出来了。Fedora15中将会在根目录中引入一个新的目录/run。而且据这位来在redhat的同学称，Fedora，Debian，Suse以及Ubuntu等发行版的开发人员已经就这件事请谈妥了。Fedora和Suse已经添加这个目录了，Debian和Ubuntu也会马上跟上。而引入这个目录的目的是为了使run-time-dir管理更加标准。

### /var/run是干什么用的

根据linux的[文件系统分层结构标准（FHS）](http://www.pathname.com/fhs/2.2/fhs-5.13.html)中的定义： 
    
    
    /var/run 目录中存放的是自系统启动以来描述系统信息的文件。
    比较常见的用途是daemon进程将自己的pid保存到这个目录。
    标准要求这个文件夹中的文件必须是在系统启动的时候清空，以便建立新的文件。
    

为了达到这个要求，linux中/var/run使用的是[tmpfs文件系统](http://en.wikipedia.org/wiki/Tmpfs)，这是一种存储在内存中的临时文件系统，当机器关闭的时候，文件系统自然就被清空了。使用df -Th命令能看到类似的输出结果: 
    
    
    文件系统    类型    容量  已用  可用 已用%% 挂载点
    none         tmpfs    990M  384K  989M   1% /var/run
    none         tmpfs    990M     0  990M   0% /var/lock
    

当然/var/run除了保存进程的pid之外也有其他的作用，比如utmp文件，就是用来记录机器的启动时间以及当前登陆用户的。 

### 为什么要使用/run代替

这是因为/var/run文件系统并不是在系统一启动就是就绪的，而在此之前已经启动的进程就先将自己的运行信息存放在/dev中，/dev同样是一种tmpfs，而且是在系统一启动就可用的。但是/dev设计的本意是为了存放设备文件的，而不是为了保存进程运行时信息的，所以为了不引起混淆，/dev中存放进程信息的文件都以"."开始命名，也就是都是隐藏文件夹。但是即便是这样，随着文件夹的数量越来越多，/dev里面也就越来越混乱，终于有人坐不住了，所以引入了替代方案，也就是 /var/run。 

### 使用/var/run有什么好处

主要就是解决了上边说的管理不一致，最终使各个发行版统一管理。最终将/var/run和/var/lock都归并到/run中。而且在也不用使用隐藏文件夹这种伎俩了，对管理员来说轻松了不少。同样/dev中也不会有不相关的内容了。 但是这种根目录上的改变肯定不是一下就能完成的，Fedora15中只是刚引入/run目录并将/var/run和/var/run/lock挂载到/run和/run/lock上，到F16的时候，/var/run和/var/run/lock就只是做为符号链接出现了。 

### 参考

[Introducing /run ](http://lwn.net/Articles/436012/)
