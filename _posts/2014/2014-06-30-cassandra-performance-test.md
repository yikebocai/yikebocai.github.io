---
layout: post
title: Cassandra性能测试
categories: tech
tags: 
- nosql
- cassandra
- performance
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
CREATE KEYSPACE test WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': '2'
};

CREATE TABLE history (
  custom_id text,
  type text,
  datas list<text>,
  PRIMARY KEY (custom_id, type)
);
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

## 测试场景--SSD硬盘本地测试

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
### 场景六：多线程并发读写

修改`main`函数，使用多线程进行读写：

```java
public static void main(String[] args) {
        // singleThreadWrite();
        // singleThreadRead();

        for (int i = 0; i < 10; i++) {
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
    }
```

测试结果如下：

```
写入： spendTime:88289,avg:0.88289,tps:1132.6439307274973
写入： spendTime:88310,avg:0.8831,tps:1132.3745895142113
写入： spendTime:88640,avg:0.8864,tps:1128.158844765343
写入： spendTime:88828,avg:0.88828,tps:1125.7711532399694
写入： spendTime:92662,avg:0.92662,tps:1079.1910383976171
写入： spendTime:93139,avg:0.93139,tps:1073.6640934517227
写入： spendTime:111506,avg:1.11506,tps:896.8127275662296
写入： spendTime:111651,avg:1.11651,tps:895.6480461437874
写入： spendTime:113402,avg:1.13402,tps:881.8186628101797
写入： spendTime:113439,avg:1.13439,tps:881.5310431156834
读取： spendTime:163112,avg:1.63112,tps:613.0756780616999
读取： spendTime:163176,avg:1.63176,tps:612.8352208658137
读取： spendTime:163215,avg:1.63215,tps:612.6887847317955
读取： spendTime:163326,avg:1.63326,tps:612.2723877398578
读取： spendTime:163374,avg:1.63374,tps:612.0924994185121
读取： spendTime:163438,avg:1.63438,tps:611.8528126873799
读取： spendTime:163492,avg:1.63492,tps:611.6507229711546
读取： spendTime:163541,avg:1.63541,tps:611.4674607590757
读取： spendTime:168620,avg:1.6862,tps:593.0494603249911
读取： spendTime:168671,avg:1.68671,tps:592.8701436524358
```

## 测试场景--机械硬盘本地测试

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

## 测试场景--机械硬盘远程测试

上面都是在本地进行的测试，而线上肯定是要进行远程调用的，但服务器都在同一个局域网，按道理影响应该不大，但实际情况却差别很大。

先按照官方推荐的线上配置对操作系统做一个优化：

```
# vi /etc/security/limits.d/ cassandra.conf
cassandra - memlock unlimited
cassandra - nofile 100000
cassandra - nproc 32768
cassandra - as unlimited

# vi /etc/security/limits.d/90- nproc.conf 
* - nproc 32768
    
# vi /etc/sysctl.conf 
vm.max_map_count = 131072

# sudo swapoff --all
```

另外，使用`LeveledCompactionStrategy`策略，读写各10个线程，测试结果如下：

```
写入： spendTime:509845,avg:5.09845,tps:196.13804195392717
写入： spendTime:509844,avg:5.09844,tps:196.13842665599674
写入： spendTime:509917,avg:5.09917,tps:196.1103473702583
写入： spendTime:510141,avg:5.10141,tps:196.02423643659301
写入： spendTime:510400,avg:5.104,tps:195.92476489028215
写入： spendTime:516141,avg:5.16141,tps:193.74550752604426
写入： spendTime:517032,avg:5.17032,tps:193.4116263596837
写入： spendTime:517107,avg:5.17107,tps:193.3835743859588
写入： spendTime:517407,avg:5.17407,tps:193.2714478157427
写入： spendTime:518086,avg:5.18086,tps:193.01814756623418
读取： spendTime:540177,avg:5.40177,tps:185.1245054861647
读取： spendTime:540224,avg:5.40224,tps:185.10839947873473
读取： spendTime:540491,avg:5.40491,tps:185.0169568040911
读取： spendTime:540581,avg:5.40581,tps:184.98615378638908
读取： spendTime:540654,avg:5.40654,tps:184.96117664902138
读取： spendTime:540683,avg:5.40683,tps:184.95125609645578
读取： spendTime:540801,avg:5.40801,tps:184.9109006825061
读取： spendTime:544050,avg:5.4405,tps:183.80663541953865
读取： spendTime:544485,avg:5.44485,tps:183.6597886075833
读取： spendTime:544777,avg:5.44777,tps:183.56134712001423
```

结果让人大跌眼镜，和本地相比慢的不可思议，查看[官方文档](http://www.datastax.com/documentation/developer/java-driver/2.0/common/drivers/reference/connectionsOptions_c.html)看到，连接本地节点时初始化时的连接数据本地默认是2远程默认是1，最大连接数本地默认是8远程默认是2，通过测试发现，调整`MaxConnectionsPerHost`基本没有影响，增大`CoreConnectionsPerHost`倒是略有提升：

```java
builder.withPoolingOptions(new PoolingOptions()
.setCoreConnectionsPerHost(HostDistance.REMOTE, 8)
.setMaxConnectionsPerHost(HostDistance.REMOTE,8));
```

```
写入： spendTime:244957,avg:4.89914,tps:204.1174573496573
写入： spendTime:245042,avg:4.90084,tps:204.04665322679378
写入： spendTime:245094,avg:4.90188,tps:204.00336197540537
写入： spendTime:245116,avg:4.90232,tps:203.98505197539123
写入： spendTime:249833,avg:4.99666,tps:200.13368930445537
写入： spendTime:250071,avg:5.00142,tps:199.94321612662003
写入： spendTime:250096,avg:5.00192,tps:199.92322947987972
写入： spendTime:250227,avg:5.00454,tps:199.81856474321316
写入： spendTime:250274,avg:5.00548,tps:199.78103998018173
写入： spendTime:250500,avg:5.01,tps:199.6007984031936
读取： spendTime:270812,avg:5.41624,tps:184.62992777277225
读取： spendTime:270865,avg:5.4173,tps:184.593801340151
读取： spendTime:273846,avg:5.47692,tps:182.58437223841136
读取： spendTime:273869,avg:5.47738,tps:182.5690384819019
读取： spendTime:273984,avg:5.47968,tps:182.4924083158141
读取： spendTime:274228,avg:5.48456,tps:182.3300319442216
读取： spendTime:274344,avg:5.48688,tps:182.25293791735922
读取： spendTime:274362,avg:5.48724,tps:182.24098089385555
读取： spendTime:274388,avg:5.48776,tps:182.22371240724814
读取： spendTime:274727,avg:5.49454,tps:181.99885704717775
```

看起来是每次读写都要进行远程IO操作极大地影响了性能，为了减少网络开销，使用了Cassandra支持批量写操作的特性，另外优化读取也使用批量读取的方式，每次读取都合并10个数据批量处理，性能有明显提升。改造后的Java代码如下：

```java
   public static void singleThreadWrite() {
        long start = System.currentTimeMillis();
        Builder builder = Cluster.builder();
        builder.withPoolingOptions(new PoolingOptions().setCoreConnectionsPerHost(HostDistance.REMOTE, 8).setMaxConnectionsPerHost(HostDistance.REMOTE,
                                                                                                                                   8));
        builder.withSocketOptions(new SocketOptions().setKeepAlive(true).setReceiveBufferSize(100 * 1024 * 1024).setTcpNoDelay(true));
        builder.addContactPoints("192.168.6.201");

        Cluster cluster = builder.build();

        Metadata metadata = cluster.getMetadata();
        System.out.printf("Connected to cluster: %s\n", metadata.getClusterName());

        int count = 50000;
        Session session = cluster.connect();
        for (int i = 0; i < count; i += 10) {
            StringBuffer sb = new StringBuffer();
            sb.append("BEGIN BATCH\n");
            for (int j = 0; j < 10; j++) {
                String customId = generateRandomIp();
                String type = "ip";
                String newContent = generateRandomContent();
                sb.append("update test.history using ttl 86400 set txns = txns +['");
                sb.append(newContent);
                sb.append("']  where type='");
                sb.append(type);
                sb.append("' and custom_id='");
                sb.append(customId);
                sb.append("';\n");
            }
            sb.append("APPLY BATCH;");
            session.execute(sb.toString());
            if (i % 10000 == 0) {
                SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
                String cur = sdf.format(new Date());
                System.out.println("[" + cur + "] write:" + i);
            }
        }
        session.close();
        cluster.close();
        long end = System.currentTimeMillis();
        long spendTime = end - start;
        System.out.println("写入： spendTime:" + spendTime + ",avg:" + ((double) spendTime / count) + ",tps:"
                           + (count / ((double) spendTime / 1000)));
    }

    public static void singleThreadRead() {
        long start = System.currentTimeMillis();
        Builder builder = Cluster.builder();
        // builder.withClusterName("test_cluster");
        builder.withLoadBalancingPolicy(LatencyAwarePolicy.builder(new RoundRobinPolicy()).build());
        builder.withProtocolVersion(2);
        builder.withQueryOptions(new QueryOptions().setConsistencyLevel(ConsistencyLevel.ONE));
        builder.withReconnectionPolicy(new ConstantReconnectionPolicy(1000));
        builder.withRetryPolicy(FallthroughRetryPolicy.INSTANCE);
        builder.withPoolingOptions(new PoolingOptions().setCoreConnectionsPerHost(HostDistance.REMOTE, 8).setMaxConnectionsPerHost(HostDistance.REMOTE,
                                                                                                                                   8));
        builder.withSocketOptions(new SocketOptions().setKeepAlive(true).setReceiveBufferSize(100 * 1024 * 1024).setTcpNoDelay(true));
        builder.addContactPoints("192.168.6.201");

        Cluster cluster = builder.build();

        Metadata metadata = cluster.getMetadata();
        System.out.printf("Connected to cluster: %s\n", metadata.getClusterName());

        Session session = cluster.connect();
        PreparedStatement ps = session.prepare("select * from test.history  where custom_id in (?,?,?,?,?,?,?,?,?,?)  and type=?");
        BoundStatement boundStatement = new BoundStatement(ps);

        int count = 50000;
        for (int i = 0; i < count; i += 10) {
            String type = "ip";
            String[] customIds = new String[10];
            for (int j = 0; j < 10; j++) {
                customIds[j] = generateRandomIp();
            }
            BoundStatement bind = boundStatement.bind(customIds[0], customIds[1], customIds[2], customIds[3],
                                                      customIds[4], customIds[5], customIds[6], customIds[7],
                                                      customIds[8], customIds[9], type);
            ResultSet results = session.execute(bind);
            for (Row row : results) {
                // System.out.println("i:" + i + "," + row.getString("custom_id"));
                row.getList("txns", String.class);
            }

            if (i % 10000 == 0) {
                SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss");
                String cur = sdf.format(new Date());
                System.out.println("[" + cur + "] read:" + i);
            }
        }
        cluster.close();
        long end = System.currentTimeMillis();
        long spendTime = end - start;
        System.out.println("读取： spendTime:" + spendTime + ",avg:" + ((double) spendTime / count) + ",tps:"
                           + (count / ((double) spendTime / 1000)));

    }
```

测试结果如下：

```
写入： spendTime:54728,avg:1.09456,tps:913.6091214734688
写入： spendTime:54806,avg:1.09612,tps:912.3088712914645
写入： spendTime:54817,avg:1.09634,tps:912.1258003903898
写入： spendTime:55021,avg:1.10042,tps:908.7439341342397
写入： spendTime:55095,avg:1.1019,tps:907.5233687267447
写入： spendTime:55143,avg:1.10286,tps:906.7334022450719
写入： spendTime:55177,avg:1.10354,tps:906.1746742302046
写入： spendTime:55206,avg:1.10412,tps:905.6986559431946
写入： spendTime:55237,avg:1.10474,tps:905.1903615330303
写入： spendTime:55241,avg:1.10482,tps:905.1248167122246
读取： spendTime:70007,avg:1.40014,tps:714.2142928564286
读取： spendTime:70020,avg:1.4004,tps:714.0816909454442
读取： spendTime:70060,avg:1.4012,tps:713.6739937196688
读取： spendTime:70118,avg:1.40236,tps:713.083658974871
读取： spendTime:70153,avg:1.40306,tps:712.7278947443444
读取： spendTime:70154,avg:1.40308,tps:712.7177352681244
读取： spendTime:70159,avg:1.40318,tps:712.6669422312176
读取： spendTime:70166,avg:1.40332,tps:712.595844141037
读取： spendTime:70181,avg:1.40362,tps:712.4435388495461
读取： spendTime:70199,avg:1.40398,tps:712.2608584167866
```

## 测试场景--MySQL测试
使用和Cassandra类似的场景，多线程读写MySQL远程数据库，一个插入线程一个更新线程一个读线程运行结果如下：

```
读取： spendTime:1038124,avg:1.038124,tps:963.2760633604463插入： spendTime:1834581,avg:1.834581,tps:545.0835912941429更新： spendTime:2805814,avg:2.805814,tps:356.4028121607491
```

读写各10个线程测试结果如下：

```
读取： spendTime:1444698,avg:4.81566,tps:207.65585610279797
读取： spendTime:1446177,avg:4.82059,tps:207.44348720799735
读取： spendTime:1446124,avg:4.820413333333334,tps:207.45108994802658
读取： spendTime:1446659,avg:4.822196666666667,tps:207.37437087800234
读取： spendTime:1446855,avg:4.82285,tps:207.3462786526639
读取： spendTime:1447650,avg:4.8255,tps:207.2324111491037
读取： spendTime:1448883,avg:4.82961,tps:207.05605628611835
读取： spendTime:1449374,avg:4.831246666666667,tps:206.9859125387926
读取： spendTime:1449440,avg:4.831466666666667,tps:206.9764874710233
读取： spendTime:1450620,avg:4.8354,tps:206.80812342308803

写入： spendTime:1794532,avg:5.981773333333333,tps:167.17450566498675
写入： spendTime:2051139,avg:6.83713,tps:146.260199820685
写入： spendTime:2052715,avg:6.842383333333333,tps:146.14790655302855
写入： spendTime:2053115,avg:6.8437166666666664,tps:146.1194331540123
写入： spendTime:2053106,avg:6.843686666666667,tps:146.12007368348247
写入： spendTime:2053670,avg:6.845566666666667,tps:146.0799446843943
写入： spendTime:2054713,avg:6.849043333333333,tps:146.0057925364759
写入： spendTime:2056037,avg:6.853456666666666,tps:145.91177104303085
写入： spendTime:2056460,avg:6.854866666666667,tps:145.88175797243807
写入： spendTime:2058106,avg:6.860353333333333,tps:145.765086929439
```

## 测试结论

* SSD性能是好于传统机械硬盘，但提升的空间并没有预期的大
* 读性能受Compcation和Row Cache影响较大，对于读敏感的业务场景最好打开
* 读写同时进行时，读性能影响较大，即使Compaction和Row Cache已调整
* Compcation对于读性能提升明显，但实际场景中不可能太频繁执行
* 写性能受影响较小，不管是硬盘介质还是数据整理方式
* 使用多线程可以线性提升整体的TPS，但写性能比读性能还是要好很多
* 尽量使用批量操作，可以减少网络IO开销，极大提升性能
* Cassandra在写入上有极大的优势，并且在高并发下性能更好