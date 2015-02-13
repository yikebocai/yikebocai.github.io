---
layout: post
title: Cassandra源代码分析之三：写入
categories: tech
tags: 
- cassandra
---

> 上一篇 [Cassandra源代码分析之二：读取](http://yikebocai.com/2015/01/cassandra-sourcecode-2-read/)分析了Cassandra的读取过程，这篇分析一下写入过程。同理，为了简单起见，只做简单的插入分析。



## 准备
确定已有如下的keyspace和table：

```sql
CREATE KEYSPACE forseti WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'} ;

CREATE TABLE forseti.mytab (
    id text PRIMARY KEY,
    age int,
    names list<text>
) WITH bloom_filter_fp_chance = 0.01
    AND caching = '{"keys":"ALL", "rows_per_partition":"NONE"}'
    AND comment = ''
    AND compaction = {'min_threshold': '4', 'class': 'org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy', 'max_threshold': '32'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';
```

然后编写一段客户端程序：

```java
import com.datastax.driver.core.Cluster;
import com.datastax.driver.core.Cluster.Builder;
import com.datastax.driver.core.Metadata;
import com.datastax.driver.core.Session;
import com.datastax.driver.core.SocketOptions;

/**
 * @author zxb 2015年1月29日 下午5:57:20
 */
public class CassandraWriteTest {

    public static void main(String[] args) {
        Builder builder = Cluster.builder();
        builder.addContactPoint("127.0.0.1");

        // socket 链接配置
        SocketOptions socketOptions = new SocketOptions().setKeepAlive(true).setConnectTimeoutMillis(5 * 10000).setReadTimeoutMillis(100000);
        builder.withSocketOptions(socketOptions);
        Cluster cluster = builder.build();
        Metadata metadata = cluster.getMetadata();
        System.out.printf("Connected to cluster: %s\n", metadata.getClusterName());

        Session session = cluster.connect();

        for (int i = 0; i < 1; i++) {
            String id = "c" + i;
            String names = "name" + i;
            String sql = "insert into forseti.mytab (id,age,names) values('" + id + "'," + i + ",['" + names + "'])";
            session.execute(sql);
        }
        cluster.close();
    }
}

```

## 写入

先来整体看下写入的调用堆栈：

```
Daemon Thread [SharedPool-Worker-1] (Suspended)    
     StorageProxy.insertLocal(Mutation, AbstractWriteResponseHandler) line: 976    
     StorageProxy.sendToHintedEndpoints(Mutation, Iterable<InetAddress>, AbstractWriteResponseHandler, String) line: 851    
     StorageProxy$2.apply(IMutation, Iterable<InetAddress>, AbstractWriteResponseHandler, String, ConsistencyLevel) line: 122    
     StorageProxy.performWrite(IMutation, ConsistencyLevel, String, StorageProxy$WritePerformer, Runnable, WriteType) line: 712    
     StorageProxy.mutate(Collection<IMutation>, ConsistencyLevel) line: 469    
     StorageProxy.mutateWithTriggers(Collection<IMutation>, ConsistencyLevel, boolean) line: 546    
     UpdateStatement(ModificationStatement).executeWithoutCondition(QueryState, QueryOptions) line: 501    
     UpdateStatement(ModificationStatement).execute(QueryState, QueryOptions) line: 487    
     QueryProcessor.processStatement(CQLStatement, QueryState, QueryOptions) line: 186    
     QueryProcessor.process(String, QueryState, QueryOptions) line: 205    
     QueryMessage.execute(QueryState) line: 117    
     Message$Dispatcher.channelRead0(ChannelHandlerContext, Message$Request) line: 373    
     Message$Dispatcher.channelRead0(ChannelHandlerContext, Object) line: 1    
     Message$Dispatcher(SimpleChannelInboundHandler<I>).channelRead(ChannelHandlerContext, Object) line: 103    
     DefaultChannelHandlerContext(AbstractChannelHandlerContext).invokeChannelRead(Object) line: 332    
     AbstractChannelHandlerContext.access$700(AbstractChannelHandlerContext, Object) line: 31    
     AbstractChannelHandlerContext$8.run() line: 323    
     Executors$RunnableAdapter<T>.call() line: 471    
     AbstractTracingAwareExecutorService$FutureTask<T>.run() line: 162    
     SEPWorker.run() line: 103    
     Thread.run() line: 744    
```

和读取一样，执行的写入SQL也`从Message$Dispatcher.channelRead0()`方法中进入，然后到`QueryProcessor.process()`时，在`注1`处根据不同的SQL语句得到不同的Statement，查询时是`SelectStatement`，而插入时`UpdateStatement`：

```java
    public ResultMessage process(String queryString, QueryState queryState, QueryOptions options)
    throws RequestExecutionException, RequestValidationException
    {
        // 注1
        ParsedStatement.Prepared p = getStatement(queryString, queryState.getClientState());
        options.prepare(p.boundNames);
        CQLStatement prepared = p.statement;
        if (prepared.getBoundTerms() != options.getValues().size())
            throw new InvalidRequestException("Invalid amount of bind variables");

        return processStatement(prepared, queryState, options);
    }
```

最终进入到`StorageProxy.insertLocal()`方法，在`注1`处调用`SEPExecutor`来启动一个`LocalMuatationRunnable`的线程。在`注2`处提请执行：

```java
    private static void insertLocal(final Mutation mutation, final AbstractWriteResponseHandler responseHandler)
    {
        mutation.retain();

// 注1
StageManager.getStage(Stage.MUTATION).maybeExecuteImmediately(new LocalMutationRunnable()
        {
            public void runMayThrow()
            {
                try
                {
                    IMutation processed = SinkManager.processWriteRequest(mutation);
                    if (processed != null)
                    {
                        // 注2
                        ((Mutation) processed).apply();
                        responseHandler.response(null);
                    }
                }
                finally
                {
                    mutation.release();
                }
            }
        });
    }
```

之后进入的`Keyspace.apply()`方法，这里做两件事情，一是把数据写入到CommitLog，如`注1`所示。二是把数据写入到Memtable，如`注2`所示。Cassandra之所以有非常高的写入速度，就是因为它只做了一个顺序的磁盘写入和内存写入，而不像MySQL这样的关系型数据要先从BTree+查找写入的位置，然后再更新BTree+索引和数据。

```java
    public void apply(Mutation mutation, boolean writeCommitLog, boolean updateIndexes)
    {
        try (OpOrder.Group opGroup = writeOrder.start())
        {
            // write the mutation to the commitlog and memtables
            final ReplayPosition replayPosition;
            if (writeCommitLog)
            {
                // 注1
                Tracing.trace("Appending to commitlog");
                replayPosition = CommitLog.instance.add(mutation);
            }
            else
            {
                // we don't need the replayposition, but grab one anyway so that it stays stack allocated.
                // (the JVM will not stack allocate if the object may be null.)
                replayPosition = CommitLog.instance.getContext();
            }

            DecoratedKey key = StorageService.getPartitioner().decorateKey(mutation.key());
            for (ColumnFamily cf : mutation.getColumnFamilies())
            {
                ColumnFamilyStore cfs = columnFamilyStores.get(cf.id());
                if (cfs == null)
                {
                    logger.error("Attempting to mutate non-existant column family {}", cf.id());
                    continue;
                }

                Tracing.trace("Adding to {} memtable", cf.metadata().cfName);
                SecondaryIndexManager.Updater updater = updateIndexes ? cfs.indexManager.updaterFor(key, opGroup) : SecondaryIndexManager.nullUpdater;
                
                // 注2
                cfs.apply(key, cf, updater, opGroup, replayPosition);
            }
        }
    }

```

先看写入CommitLog的过程，调用`CommitLog.add()`方法，在`注1`处将计算将要序列化到文件的数据大小，在`注2`处从ByteBuffer中申请分配空间，在`注3`处将数据写入DataOutputByteBuffer中，在`注4`处标记这些数据可以刷到磁盘文件了：

```java
    private Allocation add(Mutation mutation, Allocation alloc)
    {
        assert mutation != null;
        
        // 注1
        long size = Mutation.serializer.serializedSize(mutation, MessagingService.current_version);

        long totalSize = size + ENTRY_OVERHEAD_SIZE;
        if (totalSize > MAX_MUTATION_SIZE)
        {
            throw new IllegalArgumentException(String.format("Mutation of %s bytes is too large for the maxiumum size of %s",
                                                             totalSize, MAX_MUTATION_SIZE));
        }

        // 注2
        allocator.allocate(mutation, (int) totalSize, alloc);
        try
        {
            PureJavaCrc32 checksum = new PureJavaCrc32();
            final ByteBuffer buffer = alloc.getBuffer();
            DataOutputByteBuffer dos = new DataOutputByteBuffer(buffer);

            // checksummed length
            dos.writeInt((int) size);
            checksum.update(buffer, buffer.position() - 4, 4);
            buffer.putInt(checksum.getCrc());

            int start = buffer.position();
            
            // 注3
            // checksummed mutation
            Mutation.serializer.serialize(mutation, dos, MessagingService.current_version);
            checksum.update(buffer, start, (int) size);
            buffer.putInt(checksum.getCrc());
        }
        catch (IOException e)
        {
            throw new FSWriteError(e, alloc.getSegment().getPath());
        }
        finally
        {
            // 注4
            alloc.markWritten();
        }

        executor.finishWriteFor(alloc);
        return alloc;
    }

```

在执行上面`alloc.markWritten()`，AbstractCommitLogService会有一个死循环不断检测是否可以刷到磁盘，如果检测到可以刷了，就会在`注1`处将数据真正写到文件中：

```java
AbstractCommitLogService(final CommitLog commitLog, final String name, final long pollIntervalMillis)
    {
        if (pollIntervalMillis < 1)
            throw new IllegalArgumentException(String.format("Commit log flush interval must be positive: %dms", pollIntervalMillis));

        Runnable runnable = new Runnable()
        {
            public void run()
            {
                ...
                boolean run = true;
                while (run)
                {
                    try
                    {
                        // always run once after shutdown signalled
                        run = !shutdown;

                        // sync and signal
                        long syncStarted = System.currentTimeMillis();
                        
                        // 注1
                        commitLog.sync(shutdown);
                        lastSyncedAt = syncStarted;
                        syncComplete.signalAll();
                    ...
            }
        };

        thread = new Thread(runnable, name);
        thread.start();
    }

```

它的调用栈如下：

```
Thread [PERIODIC-COMMIT-LOG-SYNCER] (Suspended)	
	owns: CommitLogSegment  (id=4442)	
	DirectByteBuffer(MappedByteBuffer).force() line: 205	
	CommitLogSegment.sync() line: 322	
	CommitLog.sync(boolean) line: 173	
	AbstractCommitLogService$1.run() line: 81	
	Thread.run() line: 744	
```

CommitLog写入成功后，会将数据写入Memtable，这个过程就比较简单了，Memtable实际上是一个Map，如`注1`处所示：

```java
    public void apply(DecoratedKey key, ColumnFamily columnFamily, SecondaryIndexManager.Updater indexer,
                      OpOrder.Group opGroup, ReplayPosition replayPosition) {
        long start = System.nanoTime();

        Memtable mt = data.getMemtableFor(opGroup);
        
        //注1
        mt.put(key, columnFamily, indexer, opGroup, replayPosition);
        maybeUpdateRowCache(key);
        metric.writeLatency.addNano(System.nanoTime() - start);
    }
```

完成上述写入，实际上Cassandra就算完成写入的过程了，Client即可结束，它自己会根据策略（Memtable的大小和过期时间）来批量将数据刷到SSTable中。

## 写入SSTable

## 小结