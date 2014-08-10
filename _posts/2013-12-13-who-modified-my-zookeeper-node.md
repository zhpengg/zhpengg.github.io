---
layout: post
title: "谁修改了我的 Zookeeper 节点"
description: ""
category: "system"
tags: ["zookeeper"]
---
zookeeper 集群一共有 5 台机器组成，上边记录的是应用程序的配置信息和进度信息。 程序在完成一些阶段性工作之后会将当前进度保存到 zk 上，这样就能够在程序意外 crash 之后能够恢复上次的进度。

之后发现本来某一个进程专属的 zk 进度点被多个进程修改，表现就是当正常的进程退出之后，节点的 mtime 以及 version 值依然在变化（节点本身的值一直是 0，看不到明显变化）。

现在的问题就是找到到底是谁在修改这个节点？
## zookeeper 日志

zookeeper 运行时会有三种日志写到本地磁盘:

{% highlight bash %}

1. zookeeper.log 记录客户端的连接建立，断开等事件，文本类型
2. log.xxxx transaction log， 记录每次更新事件，二进制类型
3. snapshot.xxxx 快照文件，二进制类型

{% endhighlight %}

了解了三种日志记录的内容，就可以开始追查上边的问题了。

### 1. 找到修改者的 session

transaction log 里记录了所有的更新事件，自然从这里开始入手。 这个文件是二进制的，所以必须配合相应的解码器一起使用：`LogFormatter`, 全称是 `org.apache.zookeeper.server.LogFormatter`，工具可以自行下载。

将二进制 transaction log 转换为文本型，如下：

{% highlight bash %}
java -Xmx2048m -cp LogFormatter.jar:log4j-1.2.15.jar org.apache.zookeeper.server.LogFormatter log.81c171976 > zk.log
{% endhighlight%}

打开文本日志，虽然还是包含部分二进制内容，不过已经不影响追查问题。

{% highlight bash%}
time:11/28/13 4:50:50 PM CST session:0x34241ce25568b0a cxid:0x529d9ae2 zxid:0x81c171976 type:setData path:/dstream_1_5_4/SubPoint/cm_ctr_fortest/1/0 version:1920306 data len:16 data:
{% endhighlight %}

通过 path 定位到我们关心的节点路径，然后过滤出 type 为 setData 的操作，最后找到对应的 session。

session 表示一条客户端与 ZooKeeper 集群的一条连接，下一步就是通过 session 找到对应的客户端 IP 和端口号。

### 2. 找到机器地址和端口号

因为 zookeeper.log 是文本型的，所以直接用 session 做 key， grep 搜索就好了。 有个需要注意的是： 因为 ZooKeeper 集群是多台机器，这个 session 不一定是由那个 ZooKeeper 实例建立的，所以可能需要全部机器找一遍才能找到。

## Reference

http://zookeeper.apache.org/doc/r3.1.2/zookeeperAdmin.html

{% include JB/setup %}
