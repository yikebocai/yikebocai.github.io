---
layout: post
title: CPU Load过高问题分析和解决方案
categories: tech
tags: 
- load
- jvm
---

上个月有一天，突然发现线上服务器Load特别高，4核心CPU几乎跑满了，之前Load一般才不到1的样子，从服务器用`Top`命令查看如下：

![](http://yikebocai.com/myimg/20141118-cpu-load-high.jpeg)

确定是Java应用程序把CPU占满了，使用`top -H -p 1193`查看占用CPU比较多的线程有哪些：

![](http://yikebocai.com/myimg/20141118-show-threads-by-top.jpeg)

然后，可以使用命令`printf %x 1298`将线程号转成十六进制，在线程栈日志中查找对应的线程西征，线程栈日志可以使用`jstack -l 1193 > jstack.log`生成。比如：1298转成十六进制是0x512，搜索后可以看到是通知代理IP服务的异步线程，如下所示：

![](http://yikebocai.com/myimg/20141118-jstack-thread.jpeg)

查看这一段代码的源代码可以发现，异步线程在从本地队列中取数据时是一个死循环，但循环中间没有使用`Thread.sleep()`，线程会一直不停地运行，导致占用CPU过多，如果没有数据暂停个几毫秒问题就解决了。

其它几个占用CPU比较高的还有Json序列化、从HashMap中放置和读取数据时的线程过高，根本原因都是HashMap在多线程时存在并发问题，将之改成并发的ConcurrentHashMap问题就可以解决了。

最后的CPU监控结果如下，可以看到有明显的下降：

![](http://yikebocai.com/myimg/20141118-zabbix-load.jpeg)

上面查找占用CPU比较高的方式比较麻烦，如果能用命令直接把占用比较高的线程打印出来就完美了，幸运的是发现了阿里的前同事写的脚本，可以直接执行打印占用高的线程，非常的方便好用，[点击这里查看](https://github.com/oldratlee/useful-shells/blob/master/show-busy-java-threads.sh)。