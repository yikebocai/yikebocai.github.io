---
layout: post
title: 一个http请求背后的故事之四
categories: tech
tags: 
- http
---

## 服务器端处理

HTTP在客户端完成封包之后，通过TCP/IP协议将内容发送到服务器端，服务端通常会使用Apache、Nginx等Web服务器来处理请求，主要是做静态资源文件的处理，比如图片、JS、CSS和Html文件等。如果是动态资源，比如用户登陆，用户名和密码都是动态输入的，需要到后台数据库进行校验，无法预先生成静态Html文件，因此还需要有专门的模块来处理动态内容。而Apache本身并不支持动态资源的处理，但它有非常好的可扩展性，通过增加相应的处理模块或者调用后面的应用服务器来完成，比如像PHP这样的脚本语言，可以在Apache上增加`mod_php`模块来处理，更多模块可以参见[这里](http://httpd.apache.org/docs/2.2/mod/)。而像Java这样的编译型语言需要专门的应用服务器Tomcat、Jboss、Jetty等来做业务逻辑的处理，Apache通过HTTP或AJP协议来请求后端的服务。同时，如果后端是一个集群有多台服务器，还可以用Apache来做负载匀衡，以提高系统的处理能力。

### Apache

Apache对HTTP请求的过程大致如下：

![Apache Request Lifecycle](http://yikebocai.com/myimg/apache-request-cycle.gif)

#### 1.wait

 打开侦听端口，一般是`80`端口，等待请求。Apache使用是的`prefork`模型，和Nginx使用的是`epoll`模型不一样，是同步阻塞型的模型，每个请求会生成一个对应的进程，当然为了提高响应速度会预先生成一些进程供使用，类似线程池的机制，因此叫`prefork`。在高并发下，需要打开非常多的进程，CPU来回在进程间切换，导致Apache无法支撑大量并发，而Nginx的`epoll`模型正式解决这一问题而出现的，号称能支撑几十万的并发请求。Apache还有一种工作模型叫`worker`，每个进程对应固定的若干个线程，可以支撑更高的并发访问，占用的内存也相对较少。但一般情况`prefork`比`worker`方式更高效，因为它比较简单并且稳定，`worker`中一个线程如果崩溃很可能导致该进程下的所有线程死掉，而`prefork`模式一个进程只对应一个线程，不会存在这种问题。

#### 2.Post-Read-Request

这一个步只读取`Request Line`和`Request Header`，前者就是我们经常看到的HTTP请求的首行，使用浏览器自带的工具查看网络请求时，并不能看到原始的内容，看到是已经经过解析的请求和响应行。不过可以通过[Wireshark](http://www.wireshark.org/)这样的抓包工具来查看，样例如下：

```
GET /path HTTP/1.1\r\n
```

另外就是可以设置环境变量，比如`nokeepalived`，强制使用短连接进行通讯，或者`force-response-1.0`强制使用HTTP 1.0 协议。

HTTP请求处理的每一步，都可以放置自己的Hook来实现特别的功能，比如像处理php动态内容的`mod_php`模块就是通过Hook的方式来实现，如果在第一阶段就想做一些特殊的处理，可以放在这个地方。

####  3.URI Translation

这个阶段的主要作用就是将URI也就是上面Request Line中的`/path`映射到实际的文件路径，其中`httpd.conf`中的`DocumentRoot`定义的实际文件所在的根路径，`Alias`设置的文件路径别名都是在这一步实现的。

#### 4.Header Parsing

这个阶段进行请求头检查和解析

#### 5.Access Control

这个阶段主要用来控制访问资源，主要根据主机名、IP地址等，在`httpd.conf`中的`Deny`和`Order`就是用来设置访问控制的，比如为了安全起见，我们暴露在公网的反向代理服务器只允许在我们公司的IP访问，从而可以有限访问外部入侵。

#### 6.Authentication and Authorization

验证和授权

#### 7.MIME Type Checking

这个阶段主要检查资源文件类型，比如资源文件类型、编码格式和语言等。

#### 8.Fix-up

生成响应内容前的一个通用阶段，和Post-Read-Request阶段类似，也是最常用的放置Hook的阶段。

#### 9.Response

生成响应内容，这个是最核心的阶段

#### 10.Logging

记录请求日志

#### 11.Clean up

清理本次请求处理完成后遗留的环境，关闭Socket连接等，是处理的最后一个阶段。


### Tomcat

Tomcat首先也是建立侦听，写过Socket程序的人可能对此都不陌生，但现在很多从事Web开发的程序员，从来没有自己写过Socket程序，可能还真不太清楚，就Java程序来说，阻塞式的服务端程序大致如下：

```java
public class Server{
public static void main(String[] args){
try {  
            ServerSocket ss = new ServerSocket(80);  
            while(true){  
                Socket s = ss.accept();  
               // do something
            }  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
     }   
 }

```
    
不过阻塞式的通讯模式效率低下，如果业务逻辑处理较慢，会导致大量进程阻塞，为了性能会使用更加高效的非阻塞`NIO`模型。

**参考资料**

1. [从运行原理及使用场景看Apache和Nginx](http://yansu.org/2014/02/15/apache-and-nginx.html)
2. [Apache与Nginx网络模型](http://blog.csdn.net/xifeijian/article/details/17385831)