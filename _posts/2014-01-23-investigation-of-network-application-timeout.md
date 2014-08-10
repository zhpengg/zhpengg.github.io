---
layout: post
title: "一次网络超时的 Debug 过程"
description: ""
category: "system"
tags: ["linux", "tcp"]
---
线上有部分机器连接外部存储时经常会会发生读写超时，通过 sar 工具查看网卡流量并不是太高，千兆网卡流入流出带宽只达到一半左右：

{% highlight bash %}
$ sar -n DEV 2
01:59:39 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
01:59:41 PM        lo  55256.00  55256.00  14330.32  14330.32      0.00      0.00      0.00
01:59:41 PM      eth0      0.00      0.00      0.00      0.00      0.00      0.00      0.00
01:59:41 PM      eth1  93348.50  87195.00  34645.71  22573.81      0.00      0.00      0.00
{% endhighlight %}

通过 ifconfig 工具发现网卡有丢包！

{% highlight bash %}
eth1      Link encap:Ethernet  HWaddr 00:25:90:A6:87:4F  
          inet addr:10.xx.xx.xx  Bcast:10.xx.xx.xx  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:470738090346 errors:0 dropped:1233691485 overruns:1233691485 frame:0
          TX packets:446204648603 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:222476057867263 (202.3 TiB)  TX bytes:153653548522801 (139.7 TiB)
{% endhighlight %}

多次执行能看到丢包数一直在增加，而且 dropped 的数目和 overruns 的数目是一致的。 根据 http://www.faqs.org/docs/linux_network/x-087-2-iface.ifconfig.html 的解释

{% highlight bash %}
Receiver overruns usually occur when packets come in faster than the kernel can service the last interrupt.
{% endhighlight %}

通俗的解释就是数据包的流量超过了 CPU 的处理能力，因为每网卡收到数据包都会触发 CPU 的中断，CPU 再将数据拷贝到内核缓冲区。

我们通过 `mpstat` 工具进一步确认这个情况：
{% highlight bash %}
$ mpstat -P ALL 2
03:28:54 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
03:28:56 PM  all   43.28    0.02    8.81    0.00    0.00    5.44    0.00    0.00   42.45
03:28:56 PM    0   23.50    0.00    5.50    0.00    0.00   67.50    0.00    0.00    3.50
03:28:56 PM    1   50.75    0.00   11.06    0.00    0.00    3.02    0.00    0.00   35.18
03:28:56 PM    2   55.00    0.00   10.50    0.00    0.00    3.00    0.00    0.00   31.50
03:28:56 PM    3   46.97    0.00    9.60    0.00    0.00    2.53    0.00    0.00   40.91
03:28:56 PM    4   49.50    0.00    9.50    0.00    0.00    3.00    0.00    0.00   38.00
03:28:56 PM    5   48.00    0.00    8.50    0.00    0.00    2.00    0.00    0.00   41.50
03:28:56 PM    6   37.81    0.00   12.94    0.00    0.00    5.97    0.00    0.00   43.28
03:28:56 PM    7   36.50    0.00   12.00    0.00    0.00    5.50    0.00    0.00   46.00
03:28:56 PM    8   40.50    0.50   10.50    0.00    0.00    5.00    0.00    0.00   43.50
{% endhighlight %}

很明显 CPU0 idle 只剩下个位数了，而且绝大部分都是被软中断占用，进一步确定是 CPU 被网卡中断打满 导致网卡丢包，进一步造成应用层网络程序超时。

## 解决办法

因为网卡会没收到一个数据包就触发一次 CPU 中断，所以降低数据包的个数，也就是将大量的小包攒成打包发送能够有效降低 CPU 中断次数。

另外就是打开网卡的多中断队列，从上图我们看到网卡的中断全部都有 CPU0 处理，打开多中断队列后，其他 CPU 也能处理网卡中断，也能够减缓单个 CPU 的处理压力。

需要注意的是网卡多中断队列需要网卡驱动支持，而且启停都需要重启机器，对业务影响比较大。
{% include JB/setup %}
