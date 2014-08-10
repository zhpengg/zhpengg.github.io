---
layout: post
title: 管道与消息队列简单性能测试
description: 
category: "system"
tags: ['linux']
---

Unix/Linux中进程间通信有多种方式，比较典型的有管道，命名管道（FIFO），消息队列，信号量和共享存储区。后面三种也叫POSIX XSI IPC。《Unix环境高级编程》中，进程间通信一章对“消息队列”，“STREAMS管道”和“UNIX域套接字”三种IPC的执行时间在Solaris9上进行了比较。考虑到测试时间距离现在已经比较久远，所以本文按照《APUE》中的思路在Linux2.6上重新进行一次测试。

### 测试环境

OS：2.6.35-29-generic #51-Ubuntu SMP CPU：Pentium(R) Dual-Core E5800 @ 3.20GHz Glibc：Version 2.12.1 

### 测试方法

方法来源于《APUE》：测试程序先创建IPC通道，调用fork，然后从父进程向子进程发送约100MB数据。对于消息队列，调用10000此msgsnd，每个消息长度为2000字节。对于管道，调用100000次write，每次写2000字节。书中没有给出程序实现，自己写了一个简单的实现。上述方法稍有不同的是，我的实现中引入了磁盘IO（读取和保存文件），但对每一种IPC，磁盘IO时间近似相等，所以对实验结果不会有较大的影响。程序如下，[源文件下载](https://github.com/zhpengg/Just-4-fun/blob/master/test/ipc_benchmark.c)： 
    
    
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    #include 
    
    #define BUFSIZE 2000
    
    static void err_exit(const char *msg)
    {
        fprintf(stderr, "%s:%s
    ", msg, strerror(errno));
        exit(-1);
    }
    
    void benchmark_pipe(const char *src, const char *dst)
    {
        int pipefd[2], ret;
        char buf[2000];
        pid_t pid;
        int fdin, fdout;
    
        if (pipe(pipefd) < 0) {
            err_exit("pipe");
        }
    
        if ((pid = fork()) < 0) {
            err_exit("fork");
        } else if (pid > 0) {
            close(pipefd[0]);
    
            if ((fdin = open(src, O_RDONLY)) < 0) {
                err_exit("parent open");
            }
            while ((ret = read(fdin, buf, BUFSIZE)) >= 0) {
                if (ret == 0) {
                    break;
                }
    
                if (write(pipefd[1], buf, ret) != ret) {
                    err_exit("paretn write");
                }
            }
    
            if (ret < 0) {
                err_exit("parent read");
            }
            close(pipefd[1]);
            close(fdin);
        } else {
            close(pipefd[1]);
    
            if ((fdout = open(dst, O_WRONLY | O_CREAT | O_TRUNC)) < 0) {
                err_exit("child open");
            }
    
            while ((ret = read(pipefd[0], buf, BUFSIZE)) >= 0) {
                if (ret == 0) {
                    break;
                }
    
                if (write(fdout, buf, ret) != ret) {
                    err_exit("child write");
                }
            }
    
            if (ret < 0) {
                err_exit("child read");
            }
            close(pipefd[0]);
            close(fdout);
        }
    }
    
    struct mymsg {
        long mtype;
        char buf[BUFSIZE];
    };
    
    
    void benchmark_msg(const char *src, const char *dst)
    {
        pid_t pid;
        int msgid;
        struct mymsg msg;
        int fdin, fdout, ret;
        struct msqid_ds msqds;
    
        key_t mskey = 0x12345675;
        if ((msgid = msgget(mskey, IPC_CREAT | IPC_EXCL| 0666)) < 0) {
            err_exit("msgget");
        }
    
        if ((pid = fork()) < 0) {
            err_exit("fork");
        } else if (pid > 0) {
            if ((fdin = open(src, O_RDONLY)) < 0) {
                err_exit("parent open");
            }
    
            while ((ret = read(fdin, msg.buf, BUFSIZE)) >= 0) {
                if (ret == 0) {
                    break;
                }
                msg.mtype = 0x12345;
    
                if (msgsnd(msgid, &msg;, ret, 0) < 0) {
                    err_exit("msgsnd");
                }
            }
    
            if (ret < 0) {
                err_exit("parent msgsnd");
            }
            close(fdin);
            /*
            * delete msg queue
            */
            msgctl(msgid, IPC_RMID, &msqds;);
    
        } else {
            if ((msgid = msgget(mskey, 0)) < 0) {
                err_exit("child msgget");
            }
    
            if ((fdout = open(dst, O_WRONLY | O_CREAT | O_TRUNC)) < 0) {
                err_exit("child open");
            }
    
            while ((ret = msgrcv(msgid, &msg;, BUFSIZE, 0, 0)) >= 0) {
                if (write(fdout, msg.buf, ret) != ret) {
                    err_exit("child write");
                }
            }
    
            if (ret < 0) {
                if (errno != EIDRM)
                    err_exit("child read");
            }
            close(fdout);
        }
    }
    
    int main(int argc, char **argv)
    {
        if (argc != 4) {
            fprintf(stderr, "usage: ipc_benchmark [pipe|msg]  source destination
    ");
            exit(1);
        }
    
        if (strcasecmp("pipe", argv[1]) == 0) {
            benchmark_pipe(argv[2], argv[3]);
        } else {
            benchmark_msg(argv[2], argv[3]);
        }
    
        return 0;
    }
    

### 测试结果

ID
Real
User
Sys

消息队列#1
0m0.472s
0m0.028s
0m0.224s

消息队列#2
0m0.499s
0m0.012s
0m0.236s

消息队列#3
0m0.429s
0m0.020s
0m0.232s

管道#1
0m0.443s
0m0.020s
0m0.228s

管道#2
0m0.424s
0m0.032s
0m0.200s

管道#3
0m0.446s
0m0.016s
0m0.220s

### 结论

虽然距离书中测试的时间已经很长了，但是结论还是不变。正如作者说的，虽然当初消息队列的引入目的是提供一个比一般IPC更快的进程间通信方法，但是这种优势已经不复存在了，而且考虑到消息队列使用方式复杂以及其他一些问题，所以在新的应用程序中不推荐再使用他们了。

## Comments

**[sagi](#516 "2011-07-27 17:45:38"):** 多进程没啥意思，多线程才是王道啊

**[刺猬](#617 "2011-08-17 17:54:50"):** 楼上太绝对了 如果能用进程解决问题 就没有必要上多线程

