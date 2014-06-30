---
layout: post
title: Cassandran性能测试
categories: tech
tags: 
- nosql
- cassandra
---

DataStax官方给出了Cassandra和其它NoSQL性能测试对比，比如[这个](http://planetcassandra.org/blog/post/how-not-to-benchmark-cassandra-a-case-study/)，读写性能好几万TPS。也有其它NoSQL的[测试结果](http://blog.couchbase.com/mongodb-and-datastax-rearview-mirror)，说Cassandra性能并不出色。Cassandra的性能到底如何，还是要结合自己的实际业务场景和环境来亲自测试一下才会对它有更好的认识。


## 测试准备

测试单机情况下单线程的单独读写和同时读写性能，业务场景为反欺诈的用户行为数据读写，这个数据的特点是需要按照不同维度把数据分别存储在，比如某个IP的登陆历史记录，使用Cassandra提供的`List`数据结构，每次写入采用追加的方式，读写数据量分为100万。Cassandra官方文档有提到，它特别为SSD硬盘做了优化，使用SSD硬盘会有更好的性能，因此在一台配置SSD硬盘的Mac和一台配置机械硬盘的台式机上分别进行了测试，用于对比不同情况对读写的影响。

具体软件环境如下：

|JDK版本|Cassandra版本|Eclipse|
|----|------|----|
|JDK1.7|官方发布版 2.0.8|标准版4.3|

创建表结构如下：

```sql
CREATE TABLE history (
  custom_id text,
  type text,
  datas list<text>,
  PRIMARY KEY (custom_id, type)
) with replication={'class':'SimpleStrategy','replication_factor':1}

```

具体测试代码如下：

```java
package cn.fraudmetrix.sandbox.cassandra;

import java.util.HashMap;
import java.util.Map;

import com.alibaba.fastjson.JSON;
import com.datastax.driver.core.Cluster;
import com.datastax.driver.core.Metadata;
import com.datastax.driver.core.ResultSet;
import com.datastax.driver.core.Row;
import com.datastax.driver.core.Session;

/**
 * @author zxb 2014年6月30日 上午10:33:49
 */
public class CassandraVelocityTest {

    public static void main(String[] args) {
        singleThreadWrite();
        singleThreadRead();
    }

    public static void singleThreadWrite() {
        long start = System.currentTimeMillis();
        Cluster cluster = Cluster.builder().addContactPoints("127.0.0.1").build();
        Metadata metadata = cluster.getMetadata();
        System.out.printf("Connected to cluster: %s\n", metadata.getClusterName());

        Session session = cluster.connect();
        int count = 1000000;
        for (int i = 0; i < count; i++) {
            
            String customId = generateRandomIp();
            String type = "ip";
            String newContent = generateRandomContent();
            StringBuffer sb = new StringBuffer();
            sb.append("update test.history using ttl 86400 set txns = txns +['");
            sb.append(newContent);
            sb.append("']  where type='");
            sb.append(type);
            sb.append("' and custom_id='");
            sb.append(customId).append("'");
            session.execute(sb.toString());
            if (i % 10000 == 0) System.out.println("write:" + i);
        }
        cluster.close();
        long end = System.currentTimeMillis();
        long spendTime = end - start;
        System.out.println("写入： spendTime:" + spendTime + ",avg:" + ((double) spendTime / count) + ",tps:"
                           + (count / ((double) spendTime / 1000)) );
    }

    public static void singleThreadRead() {
        long start = System.currentTimeMillis();
        Cluster cluster = Cluster.builder().addContactPoints("127.0.0.1").build();
        Metadata metadata = cluster.getMetadata();
        System.out.printf("Connected to cluster: %s\n", metadata.getClusterName());

        Session session = cluster.connect();
        int count = 1000000;
        for (int i = 0; i < count; i++) {
            
            String customId = generateRandomIp();
            String type = "ip";
            StringBuffer sb = new StringBuffer();
            sb.append("select * from test.history  where custom_id='");
            sb.append(customId);
            sb.append("' and type='");
            sb.append(type).append("'");
            ResultSet results = session.execute(sb.toString());
            for (Row row : results) {
                row.getString("custom_id");
                row.getList("datas", String.class);
            }
            if (i % 10000 == 0) System.out.println("read:" + i);
        }
        cluster.close();
        long end = System.currentTimeMillis();
        long spendTime = end - start;
        System.out.println("读取： spendTime:" + spendTime + ",avg:" + ((double) spendTime / count) + ",tps:"
                           + (count / ((double) spendTime / 1000)));

    }

    public static String generateRandomIp() {
        StringBuffer sb = new StringBuffer();
        sb.append("172.68.");
        sb.append((int) (Math.random() * 255)).append(".");
        sb.append((int) (Math.random() * 255));
        return sb.toString();
    }

    public static String generateRandomContent() {
        Map<String, Object> map = new HashMap<String, Object>();
        int rdm = (int) (Math.random() * 10000);
        map.put("username", "user" + rdm);
        map.put("password", "pwd" + rdm);
        map.put("email", "email" + rdm + "@126.com");
        map.put("phone", "0517-8292883" + rdm);
        map.put("ip", generateRandomIp());
        map.put("source", "web" + rdm);
        map.put("address", "hangzhou wenyixilu haichuangyuan " + rdm);
        map.put("company", "comparny" + rdm);
        return JSON.toJSONString(map);
    }
}


```

## 测试场景--SSD硬盘

|机器|操作系统| CPU|内存|硬盘|
|----|------|----|----|----|
|MacBook Pro|OS X 10.9|Intel Core i7 2.4Ghz 8核|8G 1600MHz DDR3|SSD 256G|

### 场景一：使用Cassandra的默认配置

测试结果如下：

```
写入：spendTime:221322,avg:0.221322,tps:4518.3
读取：spendTime:482894,avg:0.482894,tps:2070.8
```

### 场景二：修改配置,使用读敏感的配置

使用对读请求更敏感的`LeveledCompactionStrategy`数据整理方式，并开启Row Cache。

```sql
alter table history 
with compcation={'class':'LeveledCompactionStrategy'} 
and caching='ALL';
```

测试结果显示，写入性能稍有下降，但读性能有50%左右的显著提升。不过需要注意的是，这个读的结果是在运行远写之后执行`nodetool compact`对强制Compcation之后的结果，如下所示：

```
写入：spendTime:233175,avg:0.233175,tps:4288.6
读取：spendTime:310805,avg:0.310805,tps:3225.8
```

如果写入后接入进行读测试，读性能提升并不明显，估计是因为Cassandra的更新机制是新创建一个SSTable表，这样一来一行数据就可能存在多个SSTable中，读取的时候需要多次seek和合并后才会返回真正的数据，因此会慢很多。结果如下：

```
spendTime:468295,avg:0.468295,tps:2135.4
```

### 场景三：单独测试Row Cache开启的情况

Cassandra每次执行完读请求之后，会把结果缓存到Row Cache中，下次再有读请求后可以不用访问磁盘直接返回结果，但默认是不开启的。这里把Compcation方式改为默认的'SizeTieredCompcatioinStrategy'，只开启Cache，看是否对提升读性能有明显帮助。

```sql
alter table velocity 
with compaction={'class':'SizeTieredCompactionStrategy'} 
and caching='ALL';
```

测试结果如下：

```
读取：spendTime:473045,avg:0.473045,tps:2113.9
```

如果先执行`nodetool compact`，性能会有显著提升：

```
spendTime:338236,avg:0.338236,tps:2956.5
```

### 场景四：单独使用LeveledCompcationStrategy

```sql
alter table velocity 
with compaction={'class':'LeveledCompactionStrategy'} 
and caching='KEYS_ONLY';
```

测试结果如下：

```
读取：spendTime:474977,avg:0.474977,tps:2105.3
```

如果先执行`nodetool compact`，性能会有显著提升：

```
spendTime:324884,avg:0.324884,tps:3078.0
```

### 场景五：读写混合

```sql
alter table history 
with compcation={'class':'LeveledCompactionStrategy'} 
and caching='ALL';
```

将`main`函数的内容修改如下：

```java
public static void main(String[] args) {
        // singleThreadWrite();
        // singleThreadRead();

        Thread writeThread = new Thread() {

            @Override
            public void run() {
                singleThreadWrite();
            }
        };
        writeThread.start();

        Thread readThread = new Thread() {

            @Override
            public void run() {
                singleThreadRead();
            }
        };
        readThread.start();
    }
```

同时读写时，读性能受影响很大，写性能影响并不太明显，结果如下：

```
写入： spendTime:262577,avg:0.262577,tps:3808.4
读取： spendTime:537903,avg:0.537903,tps:1859.1
```

## 测试场景--机械硬盘

|机器|操作系统| CPU|内存|硬盘|
|----|------|----|----|----|
|台式组装机|Ubuntu 14.04|Intel 2核|8G 1600MHz DDR3|1T 机械硬盘 xx转|

### 场景一：使用Cassandra的默认配置

测试结果如下：

```
写入： spendTime:251394,avg:0.251394,tps:3977.8
读取： spendTime:600811,avg:0.600811,tps:1664.4
```

### 场景二：修改配置,使用读敏感的配置

使用对读请求更敏感的`LeveledCompactionStrategy`数据整理方式，并开启Row Cache。

```sql
alter table history 
with compcation={'class':'LeveledCompactionStrategy'} 
and caching='ALL';
```

测试结果如下：

```
写入： spendTime:291139,avg:0.291139,tps:3434.7
读取： spendTime:424296,avg:0.424296,tps:2356.8
```

### 场景三：读写混合

测试结果如下：

```
写入： spendTime:399429,avg:0.399429,tps:2503.5
读取： spendTime:788635,avg:0.788635,tps:1268.0 
```

## 测试结论

* SSD性能是好于传统机械硬盘，但提升的空间并没有预期的大
* 读性能受Compcation和Row Cache影响较大，对于读敏感的业务场景最好打开
* 读写同时进行时，读性能影响较大，即使Compaction和Row Cache已调整
* Compcation对于读性能提升明显，但实际场景中不可能太频繁执行
* 写性能受影响较小，不管是硬盘介质还是数据整理方式
* 即使最差的情况，tps也比用MySQL快很多倍，并且代码更简单