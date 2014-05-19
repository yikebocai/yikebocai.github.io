---
layout: post
title: 一个http请求背后的故事之三
categories: tech
tags: 
- http
---

### 报文封装

建立TCP或TSL通道到，需要按照HTTP报文协议进行封包。协议或者报文用一个例子说明的话，就像打电话当电话接通后，你可以说中文、英文、德语等等，只要双方约定好并能识别就可以了。HTTP报文是纯文本的明文，主要包括`HTTP Head`和`HTTP Body`两部分。