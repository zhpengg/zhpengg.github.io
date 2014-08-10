---
layout: post
title: 两个有趣的函数
description: 
category: "system"
tags: ['linux']
---

最近写代码时偶然从[友东](http://ydzhang.info)同学那里学到两个函数，一个是daemon()，另一个是 sendfile()。如果知道的同学就没必要继续看了。 

### daemon

当我们在写服务器端程序的时候，为了让服务器进程长时间在后台运行，我们一般将其设置为守护进程。《APUE》中专门有一章讲了守护进程，并给了示例代码。让一个进程变成守护进程并不复杂，大致过程如下： 

  * 首先先fork()出一个子进程，父进程退出。这一步是为了保证子进程不是进程组组长。
  * 接着子进程调用setsid()，重开一个会话(session)。此时的会话脱离了控制终端。
  * 然后可以在调用一次fork()，保证进程将来也不会获得控制终端。
  * 最后将0，1，2文件描述符关闭，并且占位，保证将来不会打开。

当然这只是最基本的流程，具体应用还要考虑到其他方面，贴一份示例代码如下： 

{% highlight c %}   
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    
    void daemonize (const char *cmd)
    {
    	int i, fd0, fd1, fd2;
    	pid_t pid;
    	struct rlimit r1;
    	struct sigaction sa;
    
    	umask(0);
    
    	if (getrlimit(RLIMIT_NOFILE, &r1;) <0) {
    		perror("getrlimit");
    		exit(-1);
    	}
    
    	if ((pid = fork()) < 0) {
    		perror("fork");
    		exit(-1);
    	} else if (pid != 0) {
    		exit(0);
    	}
    
    	setsid();
    
    	sa.sa_handler = SIG_IGN;
    	sigemptyset(&sa.sa;_mask);
    	sa.sa_flags = 0;
    	if (sigaction(SIGHUP, &sa;, NULL) < 0) {
    		perror("sigaction");
    		exit(1);
    	}
    
    	if ((pid = fork()) < 0) {
    		perror("fork2");
    		exit(1);
    	} else if (pid != 0) {
    		exit(0);
    	}
    
    	if (chdir("/") < 0) {
    		perror("chdir");
    		exit(1);
    	}
    
    	if (r1.rlim_max == RLIM_INFINITY) {
    		r1.rlim_max = 1024;
    	}	
    
    	for (i = 0; i < r1.rlim_max; i++) {
    		close(i);
    	}
    
    	/* attach file descriptors to /dev/null */
    	fd0 = open("/dev/null", O_RDWR);
    	fd1 = dup(0);
    	fd2 = dup(0);
    
    	openlog(cmd, LOG_CONS, LOG_DAEMON);
    	if (fd0 != 0 || fd1 != 1 || fd2 != 2) {
    		syslog(LOG_ERR, "unexpected file descriptors %d %d %d
    ", fd0, fd1, fd2);
    		exit(1);
    	}
    	syslog(LOG_ERR, "daemon OK!");
    }
    int main ()
    {
    	daemonize("wa haha~");	
    
    	return 0;
    }
{% endhighlight %}   
   

其实我们不需要自己写上述代码，linux的api中已经有了这个函数，函数原型如下： 
    
{% highlight c %}       
    #include 
    
    int daemon(int nochdir, int noclose);
{% endhighlight %}    

实现上和APUE如出一辙，而且man page中提到，这个函数在bsd4.4中已经提供，可能是大家受APUE影响比较深，都倾向于自己实现。 

### sendfile

用一个文件传送程序举例，当程序要将本地文件发送到网络上时，进程的大致处理过程如下： 

  * 将文件中数据文件读入一个事先开辟好的缓冲区中。
  * 将缓冲区中数据写到sockfd中。
  * 重复上两步，直到发送完毕。

看似简单自然的过程中，隐藏着一个性能问题。我们知道想write，read这种函数调用，都会变成系统调用，进而内核负责完成系统调用再将结果返回给用户。以read举例，一次read调用会发生两次内核态和用户态的切换，同时发生一次内核缓冲区到用户缓冲区的数据拷贝。 以我们上边说的文件发送程序来说，进程并不对数据进程任何处理，仅仅是读取-发送。进而抽象就是file fd -- sock fd这个过程。完全可以直接在内核态直接将文件数据发送到网络套接口。而sendfile就是为完成此任务而生，有人将其成为Zero copy。其函数原型如下： 
    
{% highlight c %}   
    #include 
    
    ssize_t sendfile(int out_fd, int in_fd, off_t \*offset, size_t count);
{% endhighlight %}   

另外需要注意一点的是，在linux2.6中，函数中的参数 out_fd必须是一个socket fd，而in_fd必须是一个file fd，并且支持mmap操作。 

### 参考

[Zero Copy I: User-Mode Perspective](http://www.linuxjournal.com/article/6345)
