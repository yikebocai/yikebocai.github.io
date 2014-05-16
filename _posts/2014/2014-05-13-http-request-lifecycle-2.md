---
layout: post
title: 一个http请求背后的故事之二
categories: tech
tags: 
- http
- tcp
---

### 建立连接

通过DNS查询获取服务器的真实IP之后，就需要进行网络连接。HTTP协议是构建在TCP/IP之上，类似的还有像FTP、 SMTP、SNMP等协议，这些都是应用层的协议，而TCP/IP是传输层的协议，打个比方你在打电话时，电话网络相当于TCP/IP，而你和对方讲的语言就是HTTP，当然你可以用不同的语言和不同的人进行对话，只要对方都理解就行。基本的网络层次结构如下所示：

![tcp ip protocal](http://yikebocai.com/myimg/tcp-ip-protocol.png)

因此进行HTTP请求的第一步是通过三次握手的方式建立TCP连接，这种方式在当年C/S架构流行大家都很清楚，指定IP和端口号，建立Socket连接，创建输出输入流，然后写数据到远端。但进入Web时代后，恐怕很多开发并不是非常清楚，因为浏览器和应用服务器都把底层的事情帮你干完了。不过可以通过一些工具，比如Wireshark、Firebug等工具，来查看TCP和HTTP的原始内容，或者自己通过Java代码来模拟一下浏览器的请求，会更加深入地理解HTTP的通讯机制。

![tcp 3 handshark](http://yikebocai.com/myimg/tcpopen3way.png)

如果更深入一点，网络通讯模型中，从应用层也就是HTTP层生成原始数据，到TCP层增加TCP头，到IP层增加IP头，一直到物理层，每一层都会添加自己的消息头，到达服务端后再按相反的顺序依次解包，如下所示：

![tcp ip protocal](http://yikebocai.com/myimg/tcp_data_transfer.png)
