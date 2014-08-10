---
layout: post
title: 关于ctime的误区
description: 
tags: ['linux']
---

本文的主旨是：千万不要把文件inode中记录的ctime理解为create time！！正确的理解是 change time。 我们知道linux上常用的文件系统比如ext2，ext3等会将文件的元信息存储在inode节点中。inode包含的信息可用下边的结构体表示： 
    
{% highlight c %}       
struct stat {
   dev_t     st_dev;     /* ID of device containing file */
   ino_t     st_ino;     /* inode number */
   mode_t    st_mode;    /* protection */
   nlink_t   st_nlink;   /* number of hard links */
   uid_t     st_uid;     /* user ID of owner */
   gid_t     st_gid;     /* group ID of owner */
   dev_t     st_rdev;    /* device ID (if special file) */
   off_t     st_size;    /* total size, in bytes */
   blksize_t st_blksize; /* blocksize for file system I/O */
   blkcnt_t  st_blocks;  /* number of 512B blocks allocated */
   time_t    st_atime;   /* time of last access */
   time_t    st_mtime;   /* time of last modification */
   time_t    st_ctime;   /* time of last status change */
};
{% endhighlight %}    

要获得这些信息可以通过系统调用stat做到，其原型如下： 
    
{% highlight c %}       
   #include 
   #include 
   #include 

   int stat(const char *path, struct stat *buf);
{% endhighlight %}

我们关注的重点在struct stat的后三个与时间相关的属性上：st_atime, st_mtime, st_ctime。如注释中所说的：atime表示文件最近一次访问的时间，mtime表示最近一次修改的时间，ctime表示最近一次状态修改的时间。前两个表述都很清楚，关键是ctime的描述，什么是”状态改变“呢？man page上有更详细的解释： 
    
    
> The field st_ctime is changed by writing or by setting inode information (i.e., owner, group, link count, mode, etc.).
    

意思是说：当你写或是设置inode节点信息的时候ctime会改变，比如：修改属主，所属群，改变链接计数，修改权限等等。实际上对于一些文件，我们很少会去做比如中列出的那写操作，典型的就是应用程序的日至文件，应用程序写成什么样就是什么样了，没有人会对他们去做改变权限之类的操作。那么你是否认为这些文件的ctime属性自生成之后就不会改变了呢？ 答案显然是否定的（要不然也不会有这个标题了==!）。 在此之前我一直认为对于日至文件这种，ctime几乎可以等价于文件的创建时间（create time），尤其是在linux的文件系统中没有专门保存文件创建日期的数据结构。在我得测试中却完全不是一回事，事实是甚至在你对文件进行最基本的写操作时，这个ctime都会更新，此时等价于mtime。 如果想察看一个文件的atime，ctime，mtime可以使用下边的命令： 
    
{% highlight bash %}       
  ls -lu # 查看atime
  ls -lc # 查看ctime
  ls -l  # 查看mtime, 默认显示
{% endhighlight %}

另外也可以使用stat命令，能够显示更全的信息。输出大致如下： 

{% highlight bash %}   
    File: "access_log"
    Size: 1068      	Blocks: 8          IO Block: 4096   普通文件
    Device: 808h/2056d	Inode: 935546      Links: 1
    Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
    Access: 2011-08-18 23:25:17.946048002 +0800
    Modify: 2011-01-20 17:26:11.084973878 +0800
    Change: 2011-01-20 17:26:11.084973878 +0800
{% endhighlight %}
