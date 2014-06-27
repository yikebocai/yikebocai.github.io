---
layout: post
title: Cassandra原理介绍
categories: tech
tags: 
- nosql
- cassandra
---

>我们的反欺诈SAAS服务，提供的是基于用户行为的实时数据分析服务，既然是实时服务对性能就有非常高的要求，吞吐量(Thoughput)要能满足大量并发用户的请求，响应时间(Response Time)要足够短不能影响用户正常的业务处理。而数据分析，是建立在大数据处理的基础上，会从各个维度分析用户的行为数据，因此在用户请求时势必要记录所有的用户行为数据，这是一个非常庞大的数据量，单机是完全无法满足业务的持续增长，需要一个能够支持方便水平扩展(Horizontal Scaling)的数据库系统，而Cassandra相对传统关系数据库和其它NoSQL更能贴合这两点的要求。

[Cassandra](http://cassandra.apache.org/)最初源自Facebook，结合了[Google BigTable](http://en.wikipedia.org/wiki/Bigtable)面向列的特性和[Amazon Dynamo](http://en.wikipedia.org/wiki/Dynamo_(storage_system\) ) [分布式哈希（DHT）](http://en.wikipedia.org/wiki/Distributed_hash_table)的P2P特性于一身，具有很高的性能、可扩展性、容错、部署简单等特点。

它虽然有多的优点，但国内使用的公司貌似不多，远没有Hbase和MongoDB火，从[百度指数](http://index.baidu.com/?tpl=trend&word=hbase%2Cmongodb%2Ccassandra)上可以明显看到这三个系统在国内的热度对比。相对国内冷静的市场来说，Cassandra在国外发展的倒是如火如荼，国外这个专门对数据库进行评分的网站[DB-Engines](http://db-engines.com/en/ranking)显示Cassandra排进了前十名，比Hbase的名次要高好几位，从2013年开始有了突飞猛进的[增长](http://db-engines.com/en/ranking_trend/system/Cassandra)，目前已有超过[1500多家公司](http://planetcassandra.org/companies/)在使用Cassandra，可惜的是没有多少国内公司，只有一家做云存储的创业公司[云诺](https://www.yunio.com/)名列榜单。这也证明了网上的中文资源为何相对匮乏，我不得不找英文资料来看，倒是顺便加强了我的英文阅读能力，也算是失之东隅得之桑榆。

## 吸引我的特性

吸引我选择Cassandra作为NoSQL的原因主要有如下三点：

### 极高的读写性能

Cassandra写数据时，首先会将请求写入Commit Log以确保数据不会丢失，然后再写入内存中的Memtable，超过内存容量后再将内存中的数据刷到磁盘的SSTable，并定期异步对SSTable做数据合并(Compaction)以减少数据读取时的查询时间。因为写入操作只涉及到顺序写入和内存操作，因此有非常高的写入性能。而进行读操作时，Cassandra支持像[LevelDB](http://www.cnblogs.com/haippy/archive/2011/12/04/2276064.html)一样的实现机制，数据分层存储，将热点数据放在Memtable和相对小的SSTable中，所以能实现非常高的读性能。

### 简单的部署结构

相对Hbase等的主从结构，Cassandra是去中心化的P2P结构，所有节点完全一样没有单点，对小公司来说，我完全可以选择数据复制份数为2，先用两三台机器把Cassandra搭起来，既能保证数据的可靠性也方便今后机器的扩展，而Hbase起码得四五台机器吧。以后为了更好地支持客户可能需要在多个地方建立数据中心，而Cassandra对多数据中心的支持也很好，可以方便地部署多个数据中心，今早还看到一个俄罗斯最大电信公司的[案例](http://planetcassandra.org/blog/post/russias-largest-telecom-talks-with-at-consulting-to-choose-cassandra-over-oracle/)。另外我们的机器现在托管在一个小机房里，万一到时机器满了无法增加要考虑搬迁机房时，使用多数据中心的方式也能做到无缝迁移。

###  和Spark的结合

Cassandra作为一个底层的存储系统，能够方便地和Spark集成进行实时计算，这一点对我们的业务场景有致命的吸引力，我看到国外有很多使用案例就是用Spark+Cassandra来实现Velocity计算，比如[Ooyala](http://www.slideshare.net/planetcassandra/south-bay-cassandrealtime-analytics-using-cassandra-spark-and-shark-at-ooyala)（需自备梯子）。


## 基本架构

Cassandra没有像BigTable或Hbase那样选择中心控制节点，而选择了无中心的P2P架构，网络中的所有节点都是对等的，它们构成了一个环，节点之间通过P2P协议每秒钟交换一次数据，这样每个节点都拥有其它所有节点的信息，包括位置、状态等，如下图所示。

![Cassandra Ring](http://yikebocai.com/myimg/cassandra-ring.png)

客户端可以连接集群中的任一个节点，和客户端建立连接的节点叫协作者(coordinator)，它相当于一个代理，负责定位该次请求要发到哪些实际拥有本次请求所需数据的节点上去获取，但如何获取并返回，主要根据客户端要求的一致性级别（Consistency Level）来定，比如：ONE指只要有一个节点返回数据就可以对客户端做出响应，QUONUM指需要返回几个根据用户的配置数目，ALL指等于数据复制份数的所有节点都返回结果才能向客户端做出响应，对于数据一致性要求不是特别高的可以选择ONE，它是最快的一种方式。

Cassandra的核心组件包括：

 **Gossip**：点对点的通讯协议，用来相互交换节点的位置和状态信息。当一个节点启动时就立即本地存储Gossip信息，但当节点信息发生变化时需要清洗历史信息，比如IP改变了。通过Gossip协议，每个节点定期每秒交换它自己和它已经交换过信息的节点的数据，每个被交换的信息都有一个版本号，这样当有新数据时可以覆盖老数据，为了保证数据交换的准确性，所有的节点必须使用同一份集群列表，这样的节点又被称作seed。

**Partitioner**：负责在集群中分配数据，由它来决定由哪些节点放置第一份的copy，一般情况会使用Hash来做主键，将每行数据分布到不同的节点上，以确保集群的可扩展性。

**Replica placement strategy**：复制策略，确定哪个节点放置复制数据，以及复制的份数。

**Snitch**：定义一个网络拓扑图，用来确定如何放置复制数据，高效地路由请求。

**cassandra.yaml**：主配置文件，设置集群的初始化配置、表的缓存参数、调优参数和资源使用、超时设定、客户端连接、备份和安全

## 写请求
 
当写事件发生时，首先由Commit Log捕获写事件并持久化，保证数据的可靠性。之后数据也会被写入到内存中，叫Memtable，当内存满了之后写入数据文件，叫SSTable，它是Log-Structured Storage Table的简称。
如果客户端配置了Consistency Level是ONE，意味着只要有一个节点写入成功，就由代理节点(Coordinator)返回给客户端写入完成。当然这中间有可能会出现其它节点写入失败的情况，Cassandra自己会通过Hinted Handoff或Read Repair 或者Anti-entropy Node Repair方式保证数据最终一致性。

对于多数据中心的写入请求，Cassandra做了优化，每个数据中心选取一个Coordinator来完成它所在数据中心的数据复制，这样客户端连接的节点只需要向数据中心的一个节点转发复制请求即可，由这个数据中心的Coordinator来完成该数据中心内的数据复制。

Cassandra的存储结构类似LSM树（[Log-Structured Merge Tree](http://en.wikipedia.org/wiki/Log-structured_merge_tree)）这种结构，不像传统数据一般都使用B+树，存储引擎以追加的方式顺序写入磁盘连续存储数据，写入是可以并发写入，不像B+树一样需要加锁，写入速度非常高，LevelDB、Hbase都是使用类似的存储结构。

 Commit Log记录每次写请求的完整信息，此时并不会根据主键进行排序，而是顺序写入，这样进行磁盘操作时没有随机写导致的磁盘大量寻道操作，对于提升速度有极大的帮助，号称最快的本地数据库LevelDB也是采用这样的策略。Commit Log会在Memtable中的数据刷入SSTable后被清除掉，因此它不会占用太多磁盘空间，Cassandra的配置时也可以单独设置存储区，这为使用高性能但容量小价格昂贵的SSD硬盘存储Commit Log，使用速度度但容量大价格非常便宜的传统机械硬盘存储数据的混合布局提供了便利。

写入到Memtable时，Cassandra能够动态地为它分配内存空间，你也可以使用工具自己调整。当达到阀值后，Memtable中的数据和索引会被放到一个队列中，然后flush到磁盘，可以使用memtable_flush_queue_size参数来指定队列的长度。当进行flush时，会停止写请求。也可以使用nodetool flush工具手动刷新 数据到磁盘，重启节点之前最好进行此操作，以减少Commit Log回放的时间。为了刷新数据，会根据partition key对Memtables进行重排序，然后顺序写入磁盘。这个过程是非常快的，因为只包含Commit Log的追加和顺序的磁盘写入。
 
当memtable中的数据刷到SSTable后，Commit Log中的数据将被清理掉。每个表会包含多个Memtable和SSTable，一般刷新完毕，SSTable是不再允许写操作的。因此，一个partition一般会跨多个SSTable文件，后续通过Compaction对多个文件进行合并，以提高读写性能。

这里所述的写请求不单指Insert操作，Update操作也是如此，Cassandra对Update操作的处理和传统关系数据库完全不一样，并不立即对原有数据进行更新，而是会增加一条新的记录，后续在进行Compaction时将数据再进行合并。Delete操作也同样如此，要删除的数据会先标记为Tombstone，后续进行Compaction时再真正永久删除。

## 读请求

读取数据时，首先检查Bloom filter，每一个SSTable都有一个Bloom filter用来检查partition key是否在这个SSTable，这一步是在访问任何磁盘IO的前面就会做掉。如果存在，再检查partition key cache，然后再做如下操作：

如果在cache中能找到索引，到compression offset map中找拥有这个数据的数据块，从磁盘上取得压缩数据并返回结果集。如果在cache中找不到索引，搜索partition summary确定索引在磁盘上的大概位置，然后获取索引入口，在SSTable上执行一次单独的寻道和一个顺序的列读取操作，下面也是到compression offset map中找拥有这个数据的数据块，从磁盘上取得压缩数据并返回结果集。读取数据时会合并Memtable中缓存的数据、多个SSTable中的数据，才返回最终的结果。比如更新用户Email后，用户名、密码等还在老的SSTable中，新的EMail记录到新的SSTable中，返回结果时需要读取新老数据并进行合并。

2.0之后的Bloom filter,compression offset map,partition summary都不放在Heap中了，只有partition key cache还放在Heap中。Bloom filter增长大约1~2G每billion partitions。partition summary是partition index的样本，你可以通过index_interval来配置样本频率。compression offset map每TB增长1~3G。对数据压缩越多，就会有越多个数的压缩块，和越大compression offset table。
 
读请求(Read Request)分两种，一种是Rirect Read Request，根据客户端配置的Consistency Level读取到数据即可返回客户端结果。一种是Background Read Repair Request,除了直接请求到达的节点外，会被发送到其它复制节点，用于修复之前写入有问题的节点，保证数据最终一致性。客户端读取时，Coordinator首先联系Consistency Level定义的节点，发送请求到最快响应的复制节点上，返回请求的数据。如果有多个节点被联系，会在内存比较每个复制节点传过的数据行，如果不一致选取最近的数据（根据时间戳）返回给客户端，并在后台更新过期的复制节点，这个过程被称作Read Repair 。

下面是Consistency Level 为ONE的读取过程，Client连接到任意一个节点上，该节点向实际拥有该数据的节点发出请求，响应最快的节点数据回到Coordinator后，就将数据返回给Client。如果其它节点数据有问题，Coordinator会将最新的数据发送有问题的节点上，进行数据的修复。

![Read One](http://yikebocai.com/myimg/cassandra-read-cl-one.png)

## 数据整理（Compaction）
  
更新操作不会立即更新，这样会导致随机读写磁盘，效率不高，Cassandra会把数据顺序写入到一个新的SSTable，并打上一个时间戳以标明数据的新旧。它也不会立马做删除操作，而是用Tombstone来标记要删除的数据。Compaction时，将多个SSTable文件中的数据整合到新的SSTable文件中，当旧SSTable上的读请求一完成，会被立即删除，空余出来的空间可以重新利用。虽然Compcation没有随机的IO访问，但还是一个重量级的操作，一般在后台运行，并通过限制它的吞吐量来控制，`compaction_throughput_mb_per_sec参数可以设置，默认是16M/s。另外，如果key cache显示整理后的数据是热点数据，操作系统会把它放入到page cache里，以提升性能。它的合并的策略有以下两种：

* **SizeTieredCompactionStrategy**:每次更新不会直接更新原来的数据，这样会造成随机访问磁盘，性能不高，而是在插入或更新直接写入下一个sstable，这样是顺序写入速度非常快，适合写敏感的操作。但是，因为数据分布在多个sstable，读取时需要多次磁盘寻道，读取的性能不高。为了避免这样情况，会定期在后台将相似大小的sstable进行合并，这个合并速度也会很快，默认情况是4个sstable会合并一次，合并时如果没有过期的数据要清理掉，会需要一倍的空间，因此最坏情况需要50%的空闲磁盘。
* **LeveledCompactionStrategy**：创建固定大小默认是5M的sstable，最上面一级为L0下面为L1，下面一层是上面一层的10倍大小。这种整理策略读取非常快，适合读敏感的情况，最坏只需要10%的空闲磁盘空间，它参考了LevelDB的实现，详见[LevelDB的具体实现原理](http://www.cnblogs.com/haippy/archive/2011/12/04/2276064.html)。
    
    这里也有关于这两种方式的[详细描述](http://www.datastax.com/dev/blog/leveled-compaction-in-apache-cassandra)。
    
## 数据复制和分发

数据分发和复制通常是一起的，数据用表的形式来组织，用主键来识别应该存储到哪些节点上，行的copy称作replica。当一个集群被创建时，至少要指定如下几个配置：Virtual Nodes，Partitioner，Replication Strategy，Snitch。

数据复制策略有两种，一种是SimpleStrategy，适合一个数据中心的情况，第一份数据放在Partitioner确定的节点，后面的放在顺时针找到的节点上，它不考虑跨数据中心和机架的复制。另外一种是NetworkTopologyStargegy，第一份数据和前一种一样，第二份复制的数据放在不同的机架上，每个数据中心可以有不同数据的replicas。

Partitioner策略有三种，默认是Murmur3Partitioner，使用MurmurHash。RandomPartitioner，使用Md5 Hash。ByteOrderedPartitioner使用数据的字节进行有顺分区。Cassandra默认使用MurmurHash，这种有更高的性能。

Snitch用来确定从哪个数据中心和哪个机架上写入或读取数据,有如下几种策略：

* DynamicSnitch：监控各节点的执行情况，根据节点执行性能自动调节，大部分情况推荐使用这种配置
* SimpleSnitch：不会考虑数据库和机架的情况，当使用SimpleStategy策略时考虑使用这种情况
* RackInterringSnitch：考虑数据库中和机架
* PropertyFileSnitch：用cassandra-topology.properties文件来自定义
* GossipPropertyFileSnitch:定义一个本地的数据中心和机架，然后使用Gossip协议将这个信息传播到其它节点，对应的配置文件是cassandra-rockdc.properties

## 失败检测和修复（Failure detection and recovery）

Cassandra从Gossip信息中确认某个节点是否可用，避免客户端请求路由到一个不可用的节点，或者执行比较慢的节点，这个通过dynamic snitch可以判断出来。Cassandra不是设定一个固定值来标记失败的节点，而是通过连续的计算单个节点的网络性能、工作量、以及其它条件来确定一个节点是否失败。节点失败的原因可能是硬件故障或者网络中断等，节点的中断通常是短暂的但有时也会持续比较久的时间。节点中断并不意味着这个节点永久不可用了，因此不会永久地从网络环中去除，其它节点会定期通过Gossip协议探测该节点是否恢复正常。如果想永久的去除，可以使用nodetool手工删除。

当节点从中断中恢复过来后，它会缺少最近写入的数据，这部分数据由其它复制节点暂为保存，叫做Hinted Handoff，可以从这里进行自动恢复。但如果节点中断时间超过max_hint_window_in_ms（默认3小时）设定的值，这部分数据将会被丢弃，此时需要用nodetool repair在所有节点上手工执行数据修复，以保证数据的一致性。

## 动态扩展

Cassandra最初版本是通过[一致性Hash](http://blog.csdn.net/sparkliang/article/details/5279393)来实现节点的动态扩展的，这样的好处是每次增加或减少节点只会影响相邻的节点，但这个会带来一个问题就是造成数据不均匀，比如新增时数据都会迁移到这台机的机器上，减少时这台机器上的数据又都会迁移到相邻的机器上，而其它机器都不能分担工作，势必会造成性能问题。从1.2版本开始，Cassandra引入了虚拟节点(Virtual Nodes)的概念，为每个真实节点分配多个虚拟节点（默认是256），这些节点并不是按Hash值顺序排列，而是随机的，这样在新增或减少一个节点时，会有很多真实的节点参与数据的迁移，从而实现了负载匀衡。

## 推荐配置
* **内存**：16G~64G，越大越好，减少读取和写入磁盘的次数
* **CPU**：8Core+ ，高并发请求
* **Disk**：写commit log和flush memtable 到 sstable时需要访问磁盘，用SSD放commit log并不能提升写性能，放数据会有显著的提升但成本太高，理想的方式是使用SSD做Row Cache的二级索引，参见这个这篇文章：[Cassandra:An SSD Boosted Key-Value Store](http://www.slideshare.net/tilmann_rabl/icde2014-ca-ssandra?qid=8d2d28a6-6afe-4384-93bd-93cb789e97c3&v=qf1&b=&from_search=2) (需自备梯子)
* **RAID**：数据盘不需要做RAID，Cassandra本身就要复制多份数据到不同的机器上，并且有JBOD机制可以自动检测损坏的磁盘
* **文件系统**：XFS，64位上接近无限大，Ext4 64位上最大支持16T
* **网卡**：千兆网卡
   
更多请参考：[Apache Cassandra 2.0 Document](http://www.datastax.com/documentation/cassandra/2.0/cassandra/gettingStartedCassandraIntro.html)
