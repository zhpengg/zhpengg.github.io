---
layout: post
title: Redis Internals - Event Library 
description: 
category: "system"
tags: ['redis', 'linux']
---

这是一篇翻译文章，原文见[﻿这里](http://redis.io/topics/internals-rediseventlib)。Redis实现了它自己的事件库。事件库的实现在ae.c文件中。要弄明白Redis事件库是如何工作的最好的方法就是弄明白Redis是如何使用它的。

### 为什么需要事件库【FAQ】

Q：你期望一个网络服务器如何工作？ A：在它监听的端口等待连接的到来并且为之服务。 Q：在一个描述符上调用accept时会阻塞，你是如何处理的？ A：先保存这个描述符然后在描述符上进行非阻塞的read/write操作。 Q：为什么要进行非阻塞的read/write？ A：如果操作正阻塞在文件（Unix中socket也是看作文件的）上，那么服务器还怎么再接受其他的连接请求呢。 Q：我觉得我要在socket上做很多的非阻塞的操作，是这样吗？ A：是的，这也正是事件库为你完成的工作。 Q：那么事件库是如何工作的？ A：它使用操作系统提供的带计时器的polling方式。 Q：那么有没有什么开源的事件库能完成你描述的功能吗？ A：是的，Libevent和Libev是我能想起来的你说的那样的库。 Q：Redis使用的这样的的事件库来处理socket I/O的吗？ A：不是的，因为一些原因Redis使用了自己的事件库。 

### 初始化事件库

在redis.c中定义了initServer函数用来初始化redisServer结构体中的一些字段。其中一个字段就是Redis事件循环 el： 
    
    
    aeEventLoop *el
    

initServer函数通过调用在ae.c文件中定义的aeCreateEventLoop函数初始化server.el字段。aeEventLoop的定义如下： 
    
    
    typedef struct aeEventLoop 
    {
        int maxfd;
        long long timeEventNextId;
        aeFileEvent events[AE_SETSIZE]; /* Registered events */
        aeFiredEvent fired[AE_SETSIZE]; /* Fired events */
        aeTimeEvent *timeEventHead;
        int stop;
        void *apidata; /* This is used for polling API specific data */
        aeBeforeSleepProc *beforesleep;
    } aeEventLoop;
    

### aeCreateEventLoop

aeCreateEventLoop函数首先分配aeEventLoop结构体的空间，然后调用ae_epoll.c文件中的aeApiCreate函数。aeApiCreate函数分配aeApiState结构的空间，其中的两个字段：epfd 存放从epoll_create函数返回的epoll文件描述符；event字段是一个Linux epoll库定义的epoll_event结构类型。event字段的用处将会在后续说明。 接下来是ae.c文件中的aeCreateTimeEvent。但是在这之前，initServer函数调用anet.c文件中anetTcpServer函数，该函数返回一个_监听的文件描述符（listening descriptor）_。这个描述符默认是在*6379端口*监听。返回的_监听的文件描述符_存放在inserver.fd字段中。 

### aeCreateTimeEvent

aeCreateTimeEvent函数接收如下参数： 

  * eventLoop：也就是redis.c文件中的server.el。 
  * miliseconds：定时器过期之后距离现在的毫秒数。 
  * proc：函数指针。定时器过期之后将要调用的函数。 
  * clientData：大部分时间都是NULL。 
  * finalizerProc：当一个已计时事件从计时事件列表中被移走之前调用的函数。  initServer函数调用aeCreateTimeEvent函数来添加一个计时事件到server.el的timeEventHead字段中。timeEventHead是一个指向计时事件的链表。redis.c文件中initServer函数调用aeCreateTimeEvent函数形式如下： 
    
    
    aeCreateTimeEvent(server.el /*eventLoop*/, 1 /*milliseconds*/, serverCron /*proc*/, NULL /*clientData*/, NULL /*finalizerProc*/);
    

redis.c文件中的serverCron会进行一些列的操作来保证Redis正常运行。 

### aeCreateFileEvent

调用aeCreateFileEvent函数的目的是执行epoll_ctl系统调用，以便将由anetTcpServer函数创建的监听描述符加入到EPOLLIN事件队列中。同时将它和aeCreateEventLoop函数调用生成的epoll描述符相关连。 下边详细解释了当initServer调用aeCreateFileEvent函数时的工作，initServer传递接下来的参数给aeCreateFileEvent函数： 

  * server.el：aeCreateEventLoop建立的事件循环。epoll描述符从server.el中获得。 
  * server.fd：负责监听的描述符，同时作为访问相关的文件事件结构体的索引。 
  * AE_READABLE：标志server.fd必须被监视EPOLLIN 事件。 
  * acceptHandler：当监视的事件到达时要执行的函数。函数指针存储在 `eventLoop->events[server.fd]->rfileProc`。  这样就完成了Redis的事件循环。 

### 处理事件循环

redis.c文件中的main函数调用ae.c中的aeMain，处理前一阶段初始化完毕的事件循环。 ae.c文件中的aeMain函数调用ae.c文件中的aeProcessEvent函数，在一个循环中处理时间到期和文件事件。 

### aeProcessEvents

aeProcessEvents函数调用aeSearchNearestTimer函数来查询事件循环中最先要过期的事件。我们的事例中只有一个事件那就是通过调用aeCreateTimeEvent函数建立的。 记住，如果将超时时间设定为1毫秒，那么通过aeCreateTimeEvent函数建立的定时器事件很可能会被忽略。 和事件循环相关连的tvp结构体被传递给ae_epoll.c文件中的aeApiPoll函数。aeApiPoll函数在epoll描述符上进行epoll_wait，同时想下边描述的方式激发eventLoop->fired表。 

  * fd：根据掩码值此时准备好读/写的描述符。 
  * mask：读/写操作可以在对应的描述符上进行。  aeApiPoll函数返回准备好的文件描述符个数。现在我们总结一下，当有客户端请求到来时候，aeApiPoll函数会发现并且使用一个正在监听的描述符实体和AE——READABLE掩码激发eventLoop->fired表格。 现在aeProcessEvents函数调用redis.c文件中已经被注册为回调函数的acceptHandler函数。acceptHandler函数在正在监听的描述符上执行accpet操作，返回一个已经和客户端建立连接的描述符。createClient函数通过调用aeCreateFileEvent函数向已经连接的描述符上添加一个文件事件。如下所示： 
    
    
    if (aeCreateFileEvent(server.el, c->fd, AE_READABLE,
        readQueryFromClient, c) == AE_ERR) {
        freeClient(c);
        return NULL;
    }
    

代码中的c指的是redisClient结构类型的变量，c->fd就是已经建立连接的描述符。 然后aeProcessEvent函数调用processTimeEvents函数。 

### processTimeEvents

processTimeEvents从时间事件列表开始处eventLoop->timeEventHead开始迭代。 对于每一个到期的时间事件，processTimeEvents调用相应的已注册的回调函数。本例中仅有一个已注册的回调函数，也就是redis.c文件中的serverCron函数。这个函数返回毫秒数，指示这个回调函数过多长时间再次调用。这写更改可以通过调用aeAddMilliSeconds函数记录，而且会在下一次循环中处理。 就是这些了。 

### 参考

[﻿Redis Event Library](http://redis.io/topics/internals-rediseventlib) \---EOF
