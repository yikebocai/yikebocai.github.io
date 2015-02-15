---
layout: post
title: Cassandra源代码分析之三：写入
categories: tech
tags: 
- cassandra
---

> 上一篇 [Cassandra源代码分析之二：读取](http://yikebocai.com/2015/01/cassandra-sourcecode-2-read/) 分析了Cassandra的读取过程，这篇分析一下写入过程。同理，为了简单起见，只做简单的插入分析。



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

CommitLog写入成功后，会将数据写入Memtable，如`注1`处所示，它最终会写入到一个BTree的数据结构中：

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

完整的调用栈如下：

```
Daemon Thread [SharedPool-Worker-1] (Suspended)	
	BTree.update(Object[], Comparator<V>, Iterable<V>, int, boolean, UpdateFunction<V>) line: 186	
	AtomicBTreeColumns.addAllWithSizeDelta(ColumnFamily, MemtableAllocator, OpOrder$Group, SecondaryIndexManager$Updater) line: 196	
	Memtable.put(DecoratedKey, ColumnFamily, SecondaryIndexManager$Updater, OpOrder$Group, ReplayPosition) line: 194	
	ColumnFamilyStore.apply(DecoratedKey, ColumnFamily, SecondaryIndexManager$Updater, OpOrder$Group, ReplayPosition) line: 1059	
	Keyspace.apply(Mutation, boolean, boolean) line: 388	
	Keyspace.apply(Mutation, boolean) line: 347	
	Mutation.apply() line: 235	
	StorageProxy$7.runMayThrow() line: 985	
	StorageProxy$7(StorageProxy$LocalMutationRunnable).run() line: 2099	
	Executors$RunnableAdapter<T>.call() line: 471	
	AbstractTracingAwareExecutorService$FutureTask<T>.run() line: 162	
	SEPWorker.run() line: 103	
	Thread.run() line: 744	

```



完成上述写入，实际上Cassandra就算完成写入的过程了，Client即可结束。

## 写入SSTable

从 [Cassandra的官方文档]() 得知，Cassandra在写入CommitLog和Memtable后就结束了，它自己会根据策略（Memtable的大小和过期时间）来批量将数据刷到SSTable中。那怎么才能找到这个写入的入口呢？看Memtable类里面的方法，发现了`isExpired()`这个方法，它会根据`memtable_flush_period_in_ms`这个配置参数来判断是否需要刷到SSTable：

```java
    /**
     * @return true if this memtable is expired. Expiration time is determined by CF's memtable_flush_period_in_ms.
     */
    public boolean isExpired()
    {
        int period = cfs.metadata.getMemtableFlushPeriod();
        return period > 0 && (System.nanoTime() - creationNano >= TimeUnit.MILLISECONDS.toNanos(period));
    }
```

再搜索`isExpired()`这个方法是在哪里被调用的，找到`ColumnFamilyStore.scheduleFlush()`这个方法，在`注1`处获取当前的Memtable，并在`注2`处进行是否过期的判断，最后在`注3`处执行业务逻辑：

```java
    void scheduleFlush() {
        int period = metadata.getMemtableFlushPeriod();
        if (period > 0) {
            logger.debug("scheduling flush in {} ms", period);
            WrappedRunnable runnable = new WrappedRunnable() {

                @Override
                protected void runMayThrow() throws Exception {
                    synchronized (data) {
                        // 注1
                        Memtable current = data.getView().getCurrentMemtable();
                        // if we're not expired, we've been hit by a scheduled flush for an already flushed memtable, so
                        // ignore
                        // 注2
                        if (current.isExpired()) {
                            if (current.isClean()) {
                                // if we're still clean, instead of swapping just reschedule a flush for later
                                scheduleFlush();
                            } else {
                                // we'll be rescheduled by the constructor of the Memtable.
                                // 注3
                                forceFlush();
                            }
                        }
                    }
                }
            };
            StorageService.scheduledTasks.schedule(runnable, period, TimeUnit.MILLISECONDS);
        }
    }
```

上面的调用最后到`ColumnFamilyStore.switchMemtable()`方法，在这里会启动一个`Flush`的异步线程：

```java
    public ListenableFuture<?> switchMemtable() {
        synchronized (data) {
            logFlush();
            Flush flush = new Flush(false);
            flushExecutor.execute(flush);
            ListenableFutureTask<?> task = ListenableFutureTask.create(flush.postFlush, null);
            postFlushExecutor.submit(task);
            return task;
        }
    }
```


Flush线程会再启动一个异步线程，它的完整调用栈如下：

```
Daemon Thread [MemtableFlushWriter:32] (Suspended)	
	RandomAccessFile.write(byte[], int, int) line: 550	
	CompressedSequentialWriter.flushData() line: 128	
	CompressedSequentialWriter(SequentialWriter).flushInternal() line: 283	
	CompressedSequentialWriter(SequentialWriter).syncInternal() line: 255	
	CompressedSequentialWriter(SequentialWriter).close() line: 436	
	CompressedSequentialWriter.close() line: 256	
	SSTableWriter.close(long) line: 466	
	SSTableWriter.closeAndOpenReader(long, long) line: 427	
	SSTableWriter.closeAndOpenReader(long) line: 422	
	SSTableWriter.closeAndOpenReader() line: 417	
	Memtable$FlushRunnable.writeSortedContents(ReplayPosition, File) line: 359	
	Memtable$FlushRunnable.runWith(File) line: 314	
	Memtable$FlushRunnable(DiskAwareRunnable).runMayThrow() line: 48	
	Memtable$FlushRunnable(WrappedRunnable).run() line: 28	
	MoreExecutors$SameThreadExecutorService.execute(Runnable) line: 297	
	ColumnFamilyStore$Flush.run() line: 978	
	JMXEnabledThreadPoolExecutor(ThreadPoolExecutor).runWorker(ThreadPoolExecutor$Worker) line: 1145	
	ThreadPoolExecutor$Worker.run() line: 615	
	Thread.run() line: 744	

```

上面`period`获取的是Memtable多长时间需要将数据刷到SSTable的时间间隔，这个值有配置参数`memtable_flush_period_in_ms`决定，[官方文档中](http://www.datastax.com/documentation/cql/3.1/cql/cql_reference/tabProp.html#tabProp__table_cql_properties) 说是默认值为`0`，但你Debug并查看源代码会发现，这个值实际上是`3600s`：

```java
    private static CFMetaData newSystemMetadata(String keyspace, String cfName, String comment, CellNameType comparator)
    {
        return new CFMetaData(keyspace, cfName, ColumnFamilyType.Standard, comparator, generateLegacyCfId(keyspace, cfName))
                             .comment(comment)
                             .readRepairChance(0)
                             .dcLocalReadRepairChance(0)
                             .gcGraceSeconds(0)
                             // 注1
                             .memtableFlushPeriod(3600 * 1000);
    }
```

这个值在`conf/cassandra.yaml`中是没有的，如果你自己试图增加一行到这个文件中，想把它设置成3分钟，会发现Cassandra在启动时就会报错，说是YAML文件不合法：

```
memtable_flush_period_in_ms: 3000
```

它实际在加载时会检测配置文件中的键名是否是`Config`类的一个属性，如果不是就会报错，因此通过上面修改配置项的方式是不行的。只能通过修改源代码来实现，否则每次Debug要等1个小时实在是受不了。

到这里原来以为已找到了从Memtable刷到SSTable的源头，但是Debug时发现，只有`system.sstable_activity`表的数据，会定期刷到磁盘，而不见我们测试的`forseti.mytab`的数据。难道业务表的数据不是通过定期刷盘的方式持久化吗？业务表的数据写入SSTable实际上有两种触发方式，一种是在启动时检查如果没有刷到SSTable，会立即刷盘。另外一种是当Memtable中的数据超过最大值时，也会刷盘。这个最大值看文档是由`memtable_total_space_in_mb`这个参数决定，我们先试图修改这个参数，`cassandra.yaml`文件中找到如下内容，把这行的注释去掉并把值修改为`1`：

```
# Total memory to use for memtables.  Cassandra will flush the largest
# memtable when this much memory is used.
# If omitted, Cassandra will set it to 1/4 of the heap.
# memtable_total_space_in_mb: 2048
```

结果启动后还是报YAML文件语法错误，再到Config类中发现没有这个属性，倒是看到`memtable_heap_space_in_mb`这个属性，遂决定把这个属性添加进去试试看，结果可以正常启动：

```
memtable_heap_space_in_mb: 1
```

为了测试是不是根据大小来决定是否刷盘，Client的测试例子循环插入的条数从1开始到100再到1000，此时发现在这里可以看到mytab的写入了，说明确实是根据大小写刷盘的：

```
SSTableWriter.close(long) line: 464	
```

但它到底是从哪里触发的呢？我使用Eclipse的`Open Call Hierarchy`功能年看Flush这个类都在哪里被调用了，最终确认上一个调用栈是从`MemtableCleanerThread`类开始的：

```
 Daemon Thread [SlabPoolCleaner] (Suspended (breakpoint at line 772 in ColumnFamilyStore))	
	owns: DataTracker  (id=12009)	
	ColumnFamilyStore.switchMemtable() line: 772	
	ColumnFamilyStore.switchMemtableIfCurrent(Memtable) line: 756	
	ColumnFamilyStore$FlushLargestColumnFamily.run() line: 1038	
	MemtableCleanerThread<P>.run() line: 71
```	

它是一个死循环，不停的检测，并在注1处调用：

```java
    @Override
    public void run()
    {
        while (true)
        {
            while (!needsCleaning())
            {
                final WaitQueue.Signal signal = wait.register();
                if (!needsCleaning())
                    signal.awaitUninterruptibly();
                else
                    signal.cancel();
            }

            // 注1
            cleaner.run();
        }
    }
```

## 小结

至此，已比较清楚了插入时先写CommitLog再写入Memtable，最后从Memtable刷到SSTable的全过程。