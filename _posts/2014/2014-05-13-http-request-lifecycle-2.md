---
layout: post
title: 一个http请求背后的故事之二
categories: tech
tags: 
- http
- tcp
- https
- tsl
---

### 建立连接

通过DNS查询获取服务器的真实IP之后，就需要进行网络连接。HTTP协议是构建在TCP/IP之上，类似的还有像FTP、 SMTP、SNMP等协议，这些都是应用层的协议，而TCP/IP是传输层的协议，打个比方你在打电话时，电话网络相当于TCP/IP，而你和对方讲的语言就是HTTP，当然你可以用不同的语言和不同的人进行对话，只要对方都理解就行。基本的网络层次结构如下所示：

![tcp ip protocal](http://yikebocai.com/myimg/tcp-ip-protocol.png)

因此进行HTTP请求的第一步是通过三次握手的方式建立TCP连接，这种方式在当年C/S架构流行大家都很清楚，指定IP和端口号，建立Socket连接，创建输出输入流，然后写数据到远端。但进入Web时代后，恐怕很多开发并不是非常清楚，因为浏览器和应用服务器都把底层的事情帮你干完了。不过可以通过一些工具，比如Wireshark、Firebug等工具，来查看TCP和HTTP的原始内容，或者自己通过Java代码来模拟一下浏览器的请求，会更加深入地理解HTTP的通讯机制。

![tcp 3 handshark](http://yikebocai.com/myimg/tcpopen3way.png)

如果更深入一点，网络通讯模型中，从应用层也就是HTTP层生成原始数据，到TCP层增加TCP头，到IP层增加IP头，一直到物理层，每一层都会添加自己的消息头，到达服务端后再按相反的顺序依次解包，如下所示：

![tcp ip protocal](http://yikebocai.com/myimg/tcp_data_transfer.png)

但是HTTP是明文传输的，对于信息比较敏感的网站比如电子商务网站，这种明文传输数据包被截取和伪造的可能性就很大，比如[中间人攻击](http://en.wikipedia.org/wiki/Man-in-the-middle_attack)等，存在严重的安全问题，因此HTTPS应运而生。HTTPS是在TCP上面增加了一层[SSL/TSL](http://en.wikipedia.org/wiki/Secure_Sockets_Layer)，对明文的HTTP报文进行了加密，防止黑客攻击。

![HTTPS Hijacking](http://yikebocai.com/myimg/Hijacking-HTTPS-Sessions.png)

SSL/TSL也是建立在TCP层之上的传输层，它的目的是和服务端协商生成加密的密钥，完成之后报文的传输和HTTP完全一致，只不过一个是明文的，一个是加密后的报文。

![SSL/TSL](http://yikebocai.com/myimg/http-https.png)

SSL/TSL层客户端和服务端协商生成加密密钥的过程被称作`handshake`，首先客户端向服务端发送一个Hello的报文请求，这个请求包括用于后续生成加密密钥的随机数以及可以接受的TSL版本、加密方式等，服务器端收到后也生成一个随机数、确认使用的加密算法等，客户端收到后首先验证服务端证书，如果没有问题会从证书中取出公钥并生成第三个随机数，也叫`pre-master key`，该随机数用公钥进行加密，客户端握手通知等，双方根据预先约定的算法用三个随机数生成一个对称加密密钥。之后，双方都用这个公钥对HTTP报文进行加解密。

![TSL Handhshake](http://yikebocai.com/myimg/SSLHandshake.png)


