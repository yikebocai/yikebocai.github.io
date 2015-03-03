---
layout: post
title: Cassandra源代码分析之四：Memtable
categories: tech
tags: 
- cassandra
---

> 上一篇 [Cassandra源代码分析之二：写入](http://yikebocai.com/2015/02/cassandra-sourcecode-3-write/) 分析了Cassandra的写入过程，会先写到CommitLog，然后再写到Memtable，然后就结束了。这一章就来看看Memtable的数据结构。


Cassandra在写入或读取时，并不是直接操作磁盘文件的，就象计算机不是直接操作硬盘而先操作内存一样，这样可以大大提升应用的性能。Memtable就是Cassandra数据在内存中的表示形式。

还记得上一章中Memtable的写入是在`ColumnFamilyStore`类中：

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

在Eclipse中Debug时，最好`注1`的断点处加上条件过滤，比如可以加上`mt.toString().contains("mytab")`，因为除了写入业务数据，还有几个系统表的数据也会写入，干扰调试。

之后会进入`Memtable`类，在`注1`处申请空间，在`注2`处把内容写入到申请的空间（ByteBuffer中）：

```java
    void put(DecoratedKey key, ColumnFamily cf, SecondaryIndexManager.Updater indexer, OpOrder.Group opGroup, ReplayPosition replayPosition)
    {
        if (replayPosition != null && writeBarrier != null)
        {
            // if the writeBarrier is set, we want to maintain lastReplayPosition; this is an optimisation to avoid
            // casing it for every write, but still ensure it is correct when writeBarrier.await() completes.
            // we clone the replay position so that the object passed in does not "escape", permitting stack allocation
            replayPosition = replayPosition.clone();
            while (true)
            {
                ReplayPosition last = lastReplayPosition.get();
                if (last.compareTo(replayPosition) >= 0)
                    break;
                if (lastReplayPosition.compareAndSet(last, replayPosition))
                    break;
            }
        }

        AtomicBTreeColumns previous = rows.get(key);
// 注1
        if (previous == null)
        {
            AtomicBTreeColumns empty = cf.cloneMeShallow(AtomicBTreeColumns.factory, false);
            final DecoratedKey cloneKey = allocator.clone(key, opGroup);
            // We'll add the columns later. This avoids wasting works if we get beaten in the putIfAbsent
            previous = rows.putIfAbsent(cloneKey, empty);
            if (previous == null)
            {
                previous = empty;
                // allocate the row overhead after the fact; this saves over allocating and having to free after, but
                // means we can overshoot our declared limit.
                int overhead = (int) (cfs.partitioner.getHeapSizeOf(key.getToken()) + ROW_OVERHEAD_HEAP_SIZE);
                allocator.onHeap().allocate(overhead, opGroup);
            }
            else
            {
                allocator.reclaimer().reclaimImmediately(cloneKey);
            }
        }

//注2
        liveDataSize.addAndGet(previous.addAllWithSizeDelta(cf, allocator, opGroup, indexer));
        currentOperations.addAndGet(cf.getColumnCount() + (cf.isMarkedForDelete() ? 1 : 0) + cf.deletionInfo().rangeCount());
    }
```

进入到`AtomicBTreeColumns`类，在注1处实现Memtable的写入：

```java
    public long addAllWithSizeDelta(final ColumnFamily cm, MemtableAllocator allocator, OpOrder.Group writeOp, Updater indexer)
    {
        ColumnUpdater updater = new ColumnUpdater(this, cm.metadata, allocator, writeOp, indexer);
        DeletionInfo inputDeletionInfoCopy = null;

        while (true)
        {
            Holder current = ref;
            updater.ref = current;
            updater.reset();

            DeletionInfo deletionInfo;
            if (cm.deletionInfo().mayModify(current.deletionInfo))
            {
                if (inputDeletionInfoCopy == null)
                    inputDeletionInfoCopy = cm.deletionInfo().copy(HeapAllocator.instance);

                deletionInfo = current.deletionInfo.copy().add(inputDeletionInfoCopy);
                updater.allocated(deletionInfo.unsharedHeapSize() - current.deletionInfo.unsharedHeapSize());
            }
            else
            {
                deletionInfo = current.deletionInfo;
            }

// 注1
            Object[] tree = BTree.update(current.tree, metadata.comparator.columnComparator(), cm, cm.getColumnCount(), true, updater);

            if (tree != null && refUpdater.compareAndSet(this, current, new Holder(tree, deletionInfo)))
            {
                indexer.updateRowLevelIndexes();
                updater.finish();
                return updater.dataSize;
            }
        }
    }

```

在`BTree.update()`这行进行Debug时可以看到，插入的数据内容实际上是放到HeapByteBuffer中了，我们做几个测试如下：

```
insert into forseti.mytab2 (id,age,name,height,org) values('a1',23,'zhang',167,'fm')
//a1,23,167,zhang,fm
[97, 49, 0, 0, 0, 23, 0, 0, 0, -89, 122, 104, 97, 110, 103, 102, 109

insert into forseti.mytab2 (id,age,name,height,org) values('a1',24,'zhang1',168,'fm')
[97, 49, 0, 0, 0, 23, 0, 0, 0, -89, 122, 104, 97, 110, 103, 102, 109, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109

insert into forseti.mytab2 (id,age,name,height,org) values('a2',24,'zhang1',168,'fm')
[97, 49, 0, 0, 0, 23, 0, 0, 0, -89, 122, 104, 97, 110, 103, 102, 109, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109, 97, 50, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109

insert into forseti.mytab2 (id,age,name,height,org) values('a2',24,'zhang1',168,'fm')
[97, 49, 0, 0, 0, 23, 0, 0, 0, -89, 122, 104, 97, 110, 103, 102, 109, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109, 97, 50, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109,

insert into forseti.mytab2 (id,age,name,height,org) values('a1',24,'zhang1',168,'fm')
[97, 49, 0, 0, 0, 23, 0, 0, 0, -89, 122, 104, 97, 110, 103, 102, 109, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109, 97, 50, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109, 0, 0, 0, 24, 0, 0, 0, -88, 122, 104, 97, 110, 103, 49, 102, 109
```

从上面可以看出，会首先写入partition key的值，比如`a1`，接着写入两个整型值，最后写入两个字符串的值，如果是同一个主键，值依次写入，如果是不同的主键，会先写主键的值，然后再写入其它字段的值。