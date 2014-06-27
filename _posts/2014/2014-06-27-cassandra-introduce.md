---
layout: post
title: Cassandra使用初探
categories: tech
tags: 
- nosql
- cassandra
---

>上篇《Cassandra原理介绍》中介绍了Cassandra的基本原理，对Cassandra的特点有了基本了解，这篇介绍一下具体的使用。


## 安装配置
首先要有`JDK1.7`的支持，然后到官网下载最新的[2.0.8版本](http://cassandra.apache.org/)的安装包，直接解压到某个文件夹即可，然后切到解压目录，执行 `bin/cassandra` 启动应用，因为默认数据是存储到`/var/lib/cassandra`目录的，因此必须使用`sudo`，否则会提示没有权限操作指定目录，加上`-f`可以指定在前台运行，方便使用`CTRL-C`直接结束应用。

```
tar xzvf apache-cassandra-2.0.8-bin.tar.gz
cd apache-cassandra-2.0.8
sudo bin/cassandra -f
```

也可以到DataStax[下载RPM或DEB安装包](http://planetcassandra.org/cassandra/)，直接运行执行`sudo service cassandra start`来启动。

## 使用cqlsh
Cassandra提供了一个REPL的工具叫`cqlsh`，是使用Python写的命令行交互工具，可以很方便地进行创建keyspace、table、CRUD等各种操作。首先执行`bin/csqlsh`连接到本地节点，进入到命令行交互模式，查看当前的keyspace有哪些：

```sql
cqlsh> desc keyspaces;

system  mykeyspace  OpsCenter  system_traces
```

使用SimpleStrategy策略创建一个新的keyspace，Cassandra里的keyspace对应MySQL里的database的概念，这种策略不会区分不同的数据中心和机架，数据复制份数为2，也就是说同一份数据最多存放在两台机器上：

```sql
cqlsh> CREATE KEYSPACE testks
 WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': '2'
};

```

Cassandra中之前是没有表的概念，之前叫`Column Family`，现在这个概念逐渐被淡化，像CQL中就直接称作Table，和传统数据库中表是一个意思。但是和传统数据库表的明显区别是必须有主键，因为一个表的数据可能是分布在多个机器上的，Cassandra使用主键来做分区，常用的分区方法有`Murmur3Partioner`、`RandomPartitioner`、`ByteOrderedPartioner`，一般默认使用第一个它的效率和散列性更好。还一个非常让人振奋的特性是列支持List、Set、Map三种集合类型，不仅仅是整形、字符串、日期等基本类型了，这给很多数据存储带来极大方便，比如一个用户帐号对应多个Email地址，或者一个事件对应多个属性等，就可以分别使用List和Map来表示，并且支持对集合的追加操作，这对一些追加的场景就特别方便，比如我们在做Velocity计算时，同一个Key值往往对应多条记录，比如记录一个IP过去3个月所有的登陆信息，就可以放在List中来表示，而不用拆成多条来存储了。创建一个表如下所示：

```sql
cqlsh:testks> create table mytab (id text,List<text>) ;

cqlsh:testks> desc table mytab;
CREATE TABLE mytab (
  id text,
  values list<text>,
  PRIMARY KEY (id)
) WITH
  bloom_filter_fp_chance=0.010000 AND
  caching='KEYS_ONLY' AND
  comment='' AND
  dclocal_read_repair_chance=0.000000 AND
  gc_grace_seconds=864000 AND
  index_interval=128 AND
  read_repair_chance=0.100000 AND
  replicate_on_write='true' AND
  populate_io_cache_on_flush='false' AND
  default_time_to_live=0 AND
  speculative_retry='99.0PERCENTILE' AND
  memtable_flush_period_in_ms=0 AND
  compaction={'class': 'SizeTieredCompactionStrategy'} AND
  compression={'sstable_compression': 'LZ4Compressor'};

```

**bloom\_filter\_fp\_change**：在进行读请求操作时，Cassandra首先到`Row Cache`中查看缓存中是否有结果，如果没有会看查询的主键是否Bloom filter中，每一个SSTable都对应一个Bloom filter，以便快速确认查询结果在哪个SSTable文件中。但是共所周知，Bloom filter是有一定误差的，这个参数就是设定它的误差率。

**caching**：是否做Partition Key的缓存，它用来标明实际数据在SSTable中的物理位置。Cassandra的缓存包括Row Cache、Partitioin Key Cache，默认是开启Key Cache，如果内存足够并且有热点数据开启Row Cache会极大提升查询性能，相当于在前面加了一个Memcached。

**memtable\_flush\_period\_in\_ms**：Memtable间隔多长时间把数据刷到磁盘，实际默认情况下Memtable一般是在容量达到一定值之后会被刷到SSTable永久存储。

**compaction**：数据整理方式，Cassandra进行更新或删除操作时并不是立即对原有的旧数据进行替换或删除，这样会影响读写的性能，而是把这些操作顺序写入到一个新的SSTable中，而在定期在后台进行数据整理，把多个SSTable进行合并整理。合并的策略有SizeTieredCompactionStrategy和LeveledCompactionStrategy两种策略，前者比较适合写操作比较多的情况，后者适合读比较多的情况。

**compression**：是否对存储的数据进行压缩，一般情况下数据内容都是文本，进行压缩会节省很多磁盘空间，但会稍微消耗一些CPU时间。除了LZ4Compressor这种默认的压缩方式外，还有SnoopyCompressor等压缩方式，这种是Google发明的，号称压缩速度非常快，但压缩比一般。

插入一条数据到数据表中：

```sql
cqlsh:testks> insert into mytab(id,values) values('100',['hello','world']);
cqlsh:testks> select * from mytab;

 id  | values
-----+--------------------
 100 | ['hello', 'world']

(1 rows)
```

追加一个值到List元素中：

```sql
cqlsh:testks> update mytab set values=values+['My name is Brian'] where id='100';
cqlsh:testks> select * from mytab;

 id  | values
-----+----------------------------------------
 100 | ['hello', 'world', 'My name is Brian']

```

Cassandra还支持对某一行数据或集合中的某一个值设置过期时间，单位为秒，这对我们做一段时间内的Velocity计算非常方便：

```sql
cqlsh:testks> update mytab using ttl 60 set values=values+['Who are you'] where id='100';
cqlsh:testks> select * from mytab;

 id  | values
-----+-------------------------------------------------------
 100 | ['hello', 'world', 'My name is Brian', 'Who are you']

(1 rows)

cqlsh:testks> select * from mytab;

 id  | values
-----+----------------------------------------
 100 | ['hello', 'world', 'My name is Brian']
```
 
## 在Java中访问

cqlsh只是提供了一种方便的交互手段，实际生产中还是要在Java中访问Cassandra，之前访问Cassandra一般通过Thrift的方式，像[Hector](https://github.com/hector-client/hector)、[Astyanax](https://github.com/Netflix/astyanax)就是走的Thrift接口，而最新的[DataStax Java Driver 2.0](https://github.com/datastax/java-driver)使用了二进制的协议，并且全面支持CQL，性能更高使用更方便。Astyanax的代码看了半天，还是挺费解的，而DataStax的直接通过CQL来访问，来使用MySQL没有什么太大区别，特别容易理解。另外这些Driver也提供了负载匀衡、连接重试等高可用的功能。

下面的例子实现了一些基本的操作：

```java
package cn.fraudmetrix.sandbox.cassandra;

import com.datastax.driver.core.Cluster;
import com.datastax.driver.core.Host;
import com.datastax.driver.core.Metadata;
import com.datastax.driver.core.ResultSet;
import com.datastax.driver.core.Row;
import com.datastax.driver.core.Session;

/**
 * @author zxb 2014年6月11日 下午4:11:32
 */
public class CassandraTest {

    public static void main(String[] args) {

        Cluster cluster = Cluster.builder().addContactPoint("127.0.0.1").build();
        Metadata metadata = cluster.getMetadata();
        System.out.printf("Connected to cluster: %s\n", metadata.getClusterName());
        for (Host host : metadata.getAllHosts()) {
            System.out.printf("Datatacenter: %s; Host: %s; Rack: %s\n", host.getDatacenter(), host.getAddress(),
                              host.getRack());
        }

        Session session = cluster.connect();
        session.execute("CREATE KEYSPACE simplex WITH replication "
                        + "= {'class':'SimpleStrategy', 'replication_factor':3};");

        session.execute("CREATE TABLE simplex.songs (id uuid PRIMARY KEY,title text,album text,"
                        + "artist text,tags set<text>,data blob);");
        session.execute("CREATE TABLE simplex.playlists (id uuid,title text,album text, "
                        + "artist text,song_id uuid,PRIMARY KEY (id, title, album, artist));");

        session.execute("INSERT INTO simplex.songs (id, title, album, artist, tags) VALUES ("
                        + "756716f7-2e54-4715-9f00-91dcbea6cf50,'La Petite Tonkinoise','Bye Bye Blackbird',"
                        + "'Joséphine Baker',{'jazz', '2013'});");
        session.execute("INSERT INTO simplex.playlists (id, song_id, title, album, artist) VALUES ("
                        + "2cc9ccb7-6221-4ccb-8387-f22b6a1b354d,756716f7-2e54-4715-9f00-91dcbea6cf50,"
                        + "'La Petite Tonkinoise','Bye Bye Blackbird','Joséphine Baker');");

        ResultSet results = session.execute("SELECT * FROM simplex.playlists "
                                            + "WHERE id = 2cc9ccb7-6221-4ccb-8387-f22b6a1b354d;");
        // + "WHERE id = 2cc9ccb7-6221-4ccb-8387-f22b6a1b354d;");
        System.out.println(String.format("%-30s\t%-20s\t%-20s\n%s", "title", "album", "artist",
                                         "-------------------------------+-----------------------+--------------------"));
        for (Row row : results) {
            System.out.println(String.format("%-30s\t%-20s\t%-20s", row.getString("title"), row.getString("album"),
                                             row.getString("artist")));
        }
        System.out.println();
        session.execute("DROP KEYSPACE simplex;");
        cluster.close();
    }

}
```
