---
layout: post
title: Hacking Strings - Redis Internals
description: 
category: "system"
tags: ['redis']
---

这是一篇翻译文章，原文在[这里](http://redis.io/topics/internals-sds)。 最近在看Redis的源代码，[官方文档](http://redis.io/topics/internals)页面上有三篇对内部实现细节的分析文章，接下来会依次翻译到这里，然后再写一些其它部分的分析。我们知道Redis作为一个NoSQL类型的数据库，使用的是key作为存取对象的唯一标识，对“key”的通俗理解就是“字符串”。在Redis中字符串又分为两类：二进制安全(Binary Safe)的和非二进制安全的，关于二进制安全的描述可以参考[这里](http://en.wikipedia.org/wiki/Binary_safe)。Redis处理存储的内容时用的是二进制安全的字符串，而作为key使用的非二进制安全的。

### 正文

Redis 是一个基于键-值存储的数据库。Redis中使用字符串作为它的键，同时字符串也是“值”所使用的最基本的数据类型。当然还有更复杂的类型，比如：列表，集合，有序集合以及哈希表，不过即使是这些复杂的类型也是使用字符串来实现的。 Redis内部实现了自己的字符串类型。实现的细节包含在sds.c文件中（sds即为 Simple Dynamic Strings）。 
    
{% highlight c %}    
    struct sdshdr {
        long len;
        long free;
        char buf\[\];
    };
{% endhighlight %}

buf 存放的实际的字符串 len字段 存放了buff的长度。这个字段使得Redis取字符串长度的操作复杂度为O(1)。 free字段 存放了buff中剩余的空间。 len 和 free 字段可以看成是保存了buf 字符串数组的元信息。 

#### 新建Redis字符串

在 sds.h 中定义了一种新的叫做 sds 的数据类型，其实就是字符串指针： 
{% highlight c %}      
    typedef char *sds;
{% endhighlight %}

在sds.c中定义了新建Redis字符串指针的函数 sdsnewslen: 
    
{% highlight c %}    
    sds sdsnewlen(const void *init, size_t initlen) {
        struct sdshdr *sh;
    
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    #ifdef SDS_ABORT_ON_OOM
        if (sh == NULL) sdsOomAbort();
    #else
        if (sh == NULL) return NULL;
    #endif
        sh->len = initlen;
        sh->free = 0;
        if (initlen) {
            if (init) memcpy(sh->buf, init, initlen);
            else memset(sh->buf,0,initlen);
        }
        sh->buf[initlen] = '\0';
        return (char*)sh->buf;
    }
{% endhighlight %}

上边说过Redis字符串是struct sdshdr类型的。但是sdsnewlen函数返回的却是一个字符串指针！！ 这只是个小技巧，这里解释一下，假设我们如下用sdsnewlen函数新建一个Redis字符串： 
    
{% highlight c %}   
    sdsnewlen("redis", 5);
{% endhighlight %}

这个函数新建了一个struct sdshdr类型的变量，同时为 len，free和buf字段分配了空间。分配空间的代码如下： 
    
{% highlight c %}      
    sh = zmalloc(sizeof(struct sdshdr)+initlen+1); // initlen is length of init argument.
{% endhighlight %}

sdsnewlen成功返回之后，得到的Redis字符串大致是这个样子的： 
    
{% highlight bash %}   
    -----------
    |5|0|redis|
    -----------
    ^   ^
    sh  sh->buf 
{% endhighlight %}

sdsnewlen 函数返回给调用者的是sh->buf。 那么如果当你想释放sh所指向的Redis字符串所占用的空间时，该怎么办呢？ 此时你想要的是一个指向sh的指针，而你得到的却是指向sh->buf的指针。 那么你能够从指向sh->buf的指针得到指向sh的指针吗？ 是的，不过是指针运算而已。注意上边那个内存示意图，当我们从sh->buf的地址减去两个long型长度之后就得到了sh的地址。 而且巧合的是两个long型的长度加起来正好是struct sdshdr的长度。（注：将buf声明为`char buf[]`, 是一个针对可变长结构体普遍使用的编程技巧。） 我们来看一下sdslen函数是如何做的： 
    
{% highlight c %}      
    size_t sdslen(const sds s) {
        struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
        return sh->len;
    }
{% endhighlight %}

了解了这个小技巧之后，你就应该能理解整个sds.c文件中全部的内容了。 Redis字符串的实现隐藏在接口的后面，这个接口只接受字符串参数。而Redis字符串的用户不需要关心它到底是如何实现的，只需要把它当成字符串指针就好了。 

### 参考

[Wiki:Binary_Safe](http://en.wikipedia.org/wiki/Binary_safe)
[Hack Strings](http://redis.io/topics/internals-sds)
