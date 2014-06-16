---
layout: post
title: Apache和Nginx性能测试
categories: tech
tags: 
- performance
- apache
- nginx
---

上周一家移动电商接入我们的服务，因为访问一开始没有评估好，上线之后设备指纹的请求过多从而导致最前端的`Apache`反向代理性能急剧下降，直接影响API接口调用和后台管理系统的使用，使用Chrome查看网络请求建立连接时间超过`5s`，SSL连接时间超过`2s`，服务一段时间完全无法正常使用。幸好上周买了两台新服务器，可以很快将设备指纹的反向代理从Apache上独立出来，这样以后即使设备指纹的请求量过多，也不会影响核心的API服务和后台管理。因为`Nginx`最近风头实在太劲，号称可以直接十万以上的并发连接，因此顺便迁移到了Nginx上，一切归于平静。

虽然Nginx在业界已广泛使用有口皆碑，但没有亲自测试过还是不太放心，起码得知道它在我们的应用场景中到底比Apache强多少吧，这样以后也好心里有数，于准备了环境准备做个压测对比。测试环境如下：

* 物理机是8核8G内存SSD硬盘的MacBook Pro，上面部署了我们的API应用，不做过多的业务逻辑处理，收到请求后做个权限的校验即可返回Json结果，没有任务IO操作。
* 在VMWare Fusion上安装了64位的CentOS 6.4，分配了2核CPU和3G内存，同时安装了`Apache 2.2`和`Nginx 1.6`，都配置了反向代理，HTTP的访问方式。
* Apache使用默认配置，Nginx只设置了`worker_processor 2`，其它保持默认值。

在物理机下，使用`siege`做压测，分别测试Apache和Nginx在`500`、`1000`、`2000`并发压力下的响应时间、TPS和稳定性，为了保证稳定性，每个分别压3次，看是否有大的波动。压测命令如下：

```
siege -c 500 -r 10 "http://192.168.0.10:8081/riskService POST ip_address=10.106.1.194&partner_code=test1aaa&event_id=login&token_id=YXc988b9d03cbb733b022127c796a5f1ae&account_login=414814"
```

## 500并发

Nginx和Apache性能都非常好，TPS达到了500以上，几乎不相上下。

**nginx**

```
Transactions:               5000 hits
Availability:             100.00 %
Elapsed time:              10.27 secs
Data transferred:           0.27 MB
Response time:              0.01 secs
Transaction rate:         486.85 trans/sec
Throughput:             0.03 MB/sec
Concurrency:                3.64
Successful transactions:        5000
Failed transactions:               0
Longest transaction:            0.07
Shortest transaction:           0.00

Transactions:               5000 hits
Availability:             100.00 %
Elapsed time:               9.91 secs
Data transferred:           0.27 MB
Response time:              0.02 secs
Transaction rate:         504.54 trans/sec
Throughput:             0.03 MB/sec
Concurrency:               10.20
Successful transactions:        5000
Failed transactions:               0
Longest transaction:            1.23
Shortest transaction:           0.00

Transactions:               5000 hits
Availability:             100.00 %
Elapsed time:               9.95 secs
Data transferred:           0.27 MB
Response time:              0.01 secs
Transaction rate:         502.51 trans/sec
Throughput:             0.03 MB/sec
Concurrency:                4.35
Successful transactions:        5000
Failed transactions:               0
Longest transaction:            0.11
Shortest transaction:           0.00
```

**apache**

```
Transactions:               5000 hits
Availability:             100.00 %
Elapsed time:               9.88 secs
Data transferred:           0.27 MB
Response time:              0.01 secs
Transaction rate:         506.07 trans/sec
Throughput:             0.03 MB/sec
Concurrency:                3.27
Successful transactions:        5000
Failed transactions:               0
Longest transaction:            0.32
Shortest transaction:           0.00

Transactions:               5000 hits
Availability:             100.00 %
Elapsed time:              10.82 secs
Data transferred:           0.27 MB
Response time:              0.01 secs
Transaction rate:         462.11 trans/sec
Throughput:             0.02 MB/sec
Concurrency:                2.40
Successful transactions:        5000
Failed transactions:               0
Longest transaction:            0.07
Shortest transaction:           0.00

Transactions:               5000 hits
Availability:             100.00 %
Elapsed time:               9.75 secs
Data transferred:           0.27 MB
Response time:              0.00 secs
Transaction rate:         512.82 trans/sec
Throughput:             0.03 MB/sec
Concurrency:                1.42
Successful transactions:        5000
Failed transactions:               0
Longest transaction:            0.15
Shortest transaction:           0.00
```

## 1000并发

原以为Nginx会有令人惊喜的表现，没想到结果变成了惊吓，刚开始响应时间还非常快在零点零几，然后越来越慢，并伴随大量超时，结果惨不忍睹。而老牌的Apache表现非常稳定，连续多次压测都表现不俗。

**nginx**

```
Transactions:               7018 hits
Availability:              87.00 %
Elapsed time:              40.67 secs
Data transferred:           0.55 MB
Response time:              2.02 secs
Transaction rate:         172.56 trans/sec
Throughput:             0.01 MB/sec
Concurrency:              348.02
Successful transactions:        7018
Failed transactions:            1049
Longest transaction:           28.45
Shortest transaction:           0.00

Transactions:               9314 hits
Availability:              93.14 %
Elapsed time:              48.88 secs
Data transferred:           0.61 MB
Response time:              1.97 secs
Transaction rate:         190.55 trans/sec
Throughput:             0.01 MB/sec
Concurrency:              376.25
Successful transactions:        9314
Failed transactions:             686
Longest transaction:           26.65
Shortest transaction:           0.00

Transactions:               8212 hits
Availability:              88.91 %
Elapsed time:              44.16 secs
Data transferred:           0.60 MB
Response time:              2.64 secs
Transaction rate:         185.96 trans/sec
Throughput:             0.01 MB/sec
Concurrency:              490.69
Successful transactions:        8212
Failed transactions:            1024
Longest transaction:           28.13
Shortest transaction:           0.00
```

**apache**

```
Transactions:              10000 hits
Availability:             100.00 %
Elapsed time:              14.16 secs
Data transferred:           0.53 MB
Response time:              0.00 secs
Transaction rate:         706.21 trans/sec
Throughput:             0.04 MB/sec
Concurrency:                3.23
Successful transactions:       10000
Failed transactions:               0
Longest transaction:            0.07
Shortest transaction:           0.00

Transactions:              10000 hits
Availability:             100.00 %
Elapsed time:              15.56 secs
Data transferred:           0.53 MB
Response time:              0.00 secs
Transaction rate:         642.67 trans/sec
Throughput:             0.03 MB/sec
Concurrency:                3.12
Successful transactions:       10000
Failed transactions:               0
Longest transaction:            1.11
Shortest transaction:           0.00

Transactions:              10000 hits
Availability:             100.00 %
Elapsed time:              13.98 secs
Data transferred:           0.53 MB
Response time:              0.00 secs
Transaction rate:         715.31 trans/sec
Throughput:             0.04 MB/sec
Concurrency:                3.06
Successful transactions:       10000
Failed transactions:               0
Longest transaction:            0.06
Shortest transaction:           0.00
```

难道是Nginx的参数没有配置好的原因吗？随增加了`use epoll`和`worker_processor 4`，但结果没有太大改观。

```
Transactions:               6025 hits
Availability:              83.51 %
Elapsed time:              34.33 secs
Data transferred:           0.52 MB
Response time:              2.18 secs
Transaction rate:         175.50 trans/sec
Throughput:             0.02 MB/sec
Concurrency:              383.41
Successful transactions:        6025
Failed transactions:            1190
Longest transaction:           22.42
Shortest transaction:           0.00
```

## 2000并发

将并发请求增大到2000，再进行压测，这次Nginx表现倒非常稳定了，没有变慢没有超时，非常神奇，难到在高并发下它自己内部使用了不同的处理方式？Apache倒是一如既往的淡定，但最长响应时间倒时不太稳定。

**nginx**

```
Transactions:              20000 hits
Availability:             100.00 %
Elapsed time:              94.19 secs
Data transferred:           1.07 MB
Response time:              0.02 secs
Transaction rate:         212.34 trans/sec
Throughput:             0.01 MB/sec
Concurrency:                3.39
Successful transactions:       20000
Failed transactions:               0
Longest transaction:            0.18
Shortest transaction:           0.00

Transactions:              20000 hits
Availability:             100.00 %
Elapsed time:              96.35 secs
Data transferred:           1.07 MB
Response time:              0.02 secs
Transaction rate:         207.58 trans/sec
Throughput:             0.01 MB/sec
Concurrency:                3.76
Successful transactions:       20000
Failed transactions:               0
Longest transaction:            0.43
Shortest transaction:           0.00
```

**apache**

```
Transactions:              20000 hits
Availability:             100.00 %
Elapsed time:              91.20 secs
Data transferred:           1.07 MB
Response time:              0.00 secs
Transaction rate:         219.30 trans/sec
Throughput:             0.01 MB/sec
Concurrency:                0.60
Successful transactions:       20000
Failed transactions:               0
Longest transaction:           20.11
Shortest transaction:           0.00

Transactions:              20000 hits
Availability:             100.00 %
Elapsed time:              94.49 secs
Data transferred:           1.07 MB
Response time:              0.00 secs
Transaction rate:         211.66 trans/sec
Throughput:             0.01 MB/sec
Concurrency:                0.37
Successful transactions:       20000
Failed transactions:               0
Longest transaction:            0.05
Shortest transaction:           0.00
```

本来想再压一下3000个并发的效果，但siege无法支持提供无法分配内存，只能作罢。本次的测试并没有压测静态页面，因为我们的业务场景都是动态的，压测这个没有太大意义。从今天的压测结果来看，Nginx相比Apache并无任何优势，并且稳定性还差，不知道是不是参数配置的问题，后续优化一下配置再压测一下，看来暂时还不能草率地从Apache迁移到Nginx。

