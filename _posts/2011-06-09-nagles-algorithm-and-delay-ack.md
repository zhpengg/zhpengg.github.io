---
layout: post
title: Nagle's Algorithm and Delay Ack
description: 
category: "system"
tags: ['linux', 'tcp/ip']
---

今天看 Redis 代码，当服务端接收到来自客户端的请求，服务端要为这个客户端初始化一个 redisClient 结构来保存这个连接的全部信息，代码在 networking.c 的 createClient() 函数。因为 Redis 采用的非阻塞式IO来处理各个文件描述符，所以这个函数首先要做的就是将 accept 返回的文件描述符（fd）通过 anetNonBlock() 函数设置为非阻塞。而接下来的这个anetTcpNoDelay()是起什么作用的呢？ anetTcpNoDelay() 函数在 anet.c 文件中，函数只是对 setsockopt 做了层简单包装。代码如下： 
    
    
    int anetTcpNoDelay(char *err, int fd)
    {   
        int yes = 1;
        if (setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &yes;, sizeof(yes)) == -1)
        {
            anetSetError(err, "setsockopt TCP_NODELAY: %s", strerror(errno));
            return ANET_ERR;      
        }
        return ANET_OK;
    }   
    

可以看到整个函数就要完成一个任务：将这个已经建立的TCP连接的一个属性 TCP_NODELAY 设置为1。那么这个属性又是起神马作用的呢？这就涉及到TCP传输过程中的Nagle算法了。我们知道随着长时间的发展和演化，为了使TCP协议能够适应各种网络状况，TCP中加入了各种优化传输的机制，比如慢启动机制，快速重传机制，暂缓应答机制（Delay ACK）以及 Nagle算法等等。而如果将上述说的 TCP_NODELAY 设置为 1，就是要在传输时关闭Nagle算法。现在来看一下Nagle算法。 考虑这样一种情况，当时用telnet或是ssh与远端主机进行交互时，客户端每按下一个键都会产生一次TCP传输，考虑到TCP头和IP头都要占用20字节，而实际发送的数据只有一个字节，显然这种传输效率并不高。Nagle算法就是为了解决这种情况才引入的。Nagel算法最早在RFC896中提出，大致思想如下：Nagel算法1）要求TCP连接上最多最多只能有一个未确认的分组，并且在该分组的确认到达之前不能继续发送其他分组；2）同时TCP收集要发送的小分组，并且在确认到达时一并发出去。这个算法的总体考虑就是通过合并小的分组来减少网络传输的分组数目，进而提高网络的传输效率。 不过比较不幸的是TCP协议中还有一种机制——暂缓应答（Delay ACK）。暂缓应答的思想是当收到数据的时候并不立即发送应答，而是等待一段时间（大概是200ms，依赖系统），以便将这个应答和要发送的数据一起发送，从而避免了发送一个空的ACK数据包。当Delay ACK和Nagle同时存在的时候，在某些条件下会导致TCP的性能大幅下降。具体来说就是当进行 write-write-read 这种模式的交互时。参考文档中描述了这种性能下降的整个过程。 所以虽然不推荐，但是如果应用对性能要求很高的话，最好还是关闭Nagle算法。另一个代码片段： 
    
    
    int result = setsockopt(sock,   /* socket affected */
            IPPROTO_TCP,     /* set option at TCP level */
            TCP_NODELAY,     /* name of option */
            (char *) &flag;,  /* the cast is historical cruft */
            sizeof(int));    /* length of option value */
    if (result < 0)
        ... handle the error ...
    

### Reference

[Congestion Control in IP/TCP Internetworks RFC896](http://www.faqs.org/rfcs/rfc896.html) [Nagle's_algorithm](http://en.wikipedia.org/wiki/Nagle's_algorithm) [Nagle and Delay Ack Problem](http://www.stuartcheshire.org/papers/NagleDelayedAck/) \--EOF
