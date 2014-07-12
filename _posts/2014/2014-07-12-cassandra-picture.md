---
layout: post
title: 图解Cassandra
categories: tech
tags: 
- cassandra
---

>从网络上收集了几张Cassandra的图，比看文档要直观多了

## 部署结构
![deployment](http://yikebocai.com/myimg/cassandra-multi-datacenters.png)


## 架构总览

![architecture](http://yikebocai.com/myimg/cassandra-architecture.jpg)

## 写数据到SSTable
![read and write](http://yikebocai.com/myimg/cassandra-read-write.jpg)


![memtable to sstable](http://yikebocai.com/myimg/cassandra-memtable-to-sstable.jpg)

## 一个节点读取失败

![one failture](http://yikebocai.com/myimg/cassandra-read-cl-one-failure.png)

## 分区和复制

![partion and replic](http://yikebocai.com/myimg/cassandra-partition-replication.png)

![multi dc partion and replic](http://yikebocai.com/myimg/cassandra-Multi-DCReplication.png)

