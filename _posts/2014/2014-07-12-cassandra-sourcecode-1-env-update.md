---
layout: post
title: Cassandra源代码学习之一-Eclipse环境准备和数据更新
categories: tech
tags: 
- cassandra
---

>最近几周花时间把Cassandra的几个官方文档包括[Cassandra 2.0](http://www.datastax.com/documentation/cassandra/2.0/cassandra/gettingStartedCassandraIntro.html)、[CQL for Cassandra 2.0](http://www.datastax.com/documentation/cql/3.1/cql/cql_intro_c.html)、[DataStax Java Driver 2.0](http://www.datastax.com/documentation/developer/java-driver/2.0/java-driver/whatsNew2.html) 都读了一遍，也阅读了很多官方文档及Slides，也加入了邮件列表，在Twitter关注很多相关帐号，又做了一些测试和使用，对Cassandra算是有了初步的了解，为了能进一步理解Cassandra的内部机制，现在开始阅读它的源代码，试图把它的一些基本原理搞清楚，以便今后更好地使用它。

## 环境准备

首先从Github上clone Cassandra最新的代码仓库到本地：

```
git clone git@github.com:apache/cassandra.git
```

然后进入到cassandra目录：

```
cd cassandra
cp build.properties.default build.properties
```

把默认的中央仓库改成oschina的，这样在下载jar包时会快一点：

```
artifact.remoteRepository.central:     http://maven.oschina.net/content/groups/public/
```

Cassandra的编译用的是Ant，因此如果没有安装的话先要安装Ant，然后执行下面的命令生成Eclipse工程：

```
ant build
ant generate-eclipse-files
```

如果执行出如下错误，说明maven配置中`${user.home}`变量没有生效，需要手动改成绝对目录:

```
[artifact:dependencies] [WARNING] Unable to get resource 'junit:junit:pom:4.6' from repository central (http://maven.oschina.net/content/groups/public/): Specified destination directory cannot be created: /.m2/repository/junit/junit/4.6
```

Eclipse启动后导入Cassandra工程，然后在VM参数中指定`cassandra.yaml`主配置文件的路径、log4j日志文件的路径以及JVM的Heap大小，如下所示：

```
-Dcassandra.config=file:/Users/zxb/work/cassandra/conf/cassandra.yaml
-Dcassandra-foreground
-ea -Xmx1G
-Dlog4j.configuration=file:/Users/zxb/work/cassandra/conf/log4j-server.properties
```

## 入口

Cassandra不依赖Web容器，直接通过`main`函数启动，想要找到它的入口可以执行如下shell命令：

```bash
 find .|grep '\.java$'|grep src|grep -v examples|grep -v tools|xargs grep 'static void main'
```

显示结果如下：

```
./src/java/org/apache/cassandra/cli/CliMain.java:    public static void main(String args[]) throws IOException
./src/java/org/apache/cassandra/gms/FailureDetector.java:    public static void main(String[] args)
./src/java/org/apache/cassandra/service/CassandraDaemon.java:    public static void main(String[] args)
```

从名称上就可以看出来，入口类是`CassandraDaemon`。


## 更新步骤

CassandraDaemon启动后，首先会执行`setup()`方法进行初始化设置，比如JVM的检查、数据目录文件的检查、存储引擎初始化，如有必要还要做Commit Log的日志回放等，并在最后调用在`start()`方法中调用 `nativeServer.start()`和`thriftServer.start()`方法来启动本地服务和Thrift服务。前者通过Thrift接口进行通讯，比如像Cassandra自带的cqlsh，以及Java驱动Hector、Asyantax等都是通过这种协议来进行远程访问，好处是跨平台但缺点就是可能性能不太高，后者通过cql原生二进制协议来通讯，使用了高性能的Netty框架，比如官方的驱动程序DataStax java driver 2.0就是采用了后一种。官方的说明文档中明确表达最好使用后者，有更好的性能。

  

```java
public static void main(String[] args)
    {
        instance.activate();
    }
    

 public void activate()
    {
        ...
            
        setup();

        ...

        start();
        ...
    }


protected void setup(){

...

 // Thift
        InetAddress rpcAddr = DatabaseDescriptor.getRpcAddress();
        int rpcPort = DatabaseDescriptor.getRpcPort();
        int listenBacklog = DatabaseDescriptor.getRpcListenBacklog();
        thriftServer = new ThriftServer(rpcAddr, rpcPort, listenBacklog);

        // Native transport
        InetAddress nativeAddr = DatabaseDescriptor.getRpcAddress();
        int nativePort = DatabaseDescriptor.getNativeTransportPort();
        nativeServer = new org.apache.cassandra.transport.Server(nativeAddr, nativePort);
    }

public void start()
    {
        String nativeFlag = System.getProperty("cassandra.start_native_transport");
        if ((nativeFlag != null && Boolean.parseBoolean(nativeFlag)) || (nativeFlag == null && DatabaseDescriptor.startNativeTransport()))
            nativeServer.start();
        else
            logger.info("Not starting native transport as requested. Use JMX (StorageService->startNativeTransport()) or nodetool (enablebinary) to start it");

        String rpcFlag = System.getProperty("cassandra.start_rpc");
        if ((rpcFlag != null && Boolean.parseBoolean(rpcFlag)) || (rpcFlag == null && DatabaseDescriptor.startRpc()))
            thriftServer.start();
        else
            logger.info("Not starting RPC server as requested. Use JMX (StorageService->startRPCServer()) or nodetool (enablethrift) to start it");
    }
```

为了简单起见，只看Trift接口，先暂时忽略Netty接口，因为使用cqlsh测试比较方便。

在`ThriftServer.start()`方法中，使用一个线程包装CassandraServer，来接收Thrift请求，如下所示：

```java
    public void start()
    {
        if (server == null)
        {
            CassandraServer iface = getCassandraServer();
            server = new ThriftServerThread(address, port, backlog, getProcessor(iface), getTransportFactory());
            server.start();
        }
    }
```

等待Cassandra启动完毕后，可以启动cqlsh，创建一个KeySpace和Table，然后执行一条insert语句来看一下它的处理过程是什么样的。

```sql
CREATE KEYSPACE testks WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': '1'
};

USE testks;

CREATE TABLE tab1 (
  id text,
  name text,
  PRIMARY KEY (id)
)

insert into tab1(id,name) values('100','yikebocai');

```

当执行insert语句后，CassandraServer的`execute_cql3_query()`会通过Thrift接口接收到cql请求，通过debug可以看到就是我们在cqlsh终端输入的那样insert语句。至于CassandraServer是如何通过Thrift接口来接收数据并直接进入到这个方法中的，以后再专门研究。

接入到的cql请求是一个ByteBuffer，有可能在客户端指定了压缩算法和ConsistencyLevel，默认情况下是不用压缩算法，ConsistencyLevel是ONE。通过`uncompress()`方法还原cql语句，然后执行QueryProcessor类中的`process()`方法得到执行结果，当然这里是insert语句，因此返回结果为空，如果是select语法的话就是返回的结果集。

```java
public CqlResult execute_cql3_query(ByteBuffer query, Compression compression, ConsistencyLevel cLevel) throws TException
    {
        try
        {
            String queryString = uncompress(query, compression);
            if (startSessionIfRequested())
            {
                Tracing.instance.begin("execute_cql3_query",
                                       ImmutableMap.of("query", queryString));
            }
            else
            {
                logger.debug("execute_cql3_query");
            }

            ThriftClientState cState = state();
            return cState.getCQLQueryHandler().process(queryString, cState.getQueryState(), QueryOptions.fromProtocolV2(ThriftConversion.fromThrift(cLevel), Collections.<ByteBuffer>emptyList())).toThriftResult();
        }
        catch (RequestExecutionException e)
        {
            throw ThriftConversion.rethrow(e);
        }
        catch (RequestValidationException e)
        {
            throw ThriftConversion.toThrift(e);
        }
        finally
        {
            Tracing.instance.stopSession();
        }
    }
```

`process()`方法相对简单一点，将cql语句包装成一个Prepared类，然后得到要执行CqlStatement，它是UpdateStatement的实例，其它的还SelectStatement等。

```java
    public ResultMessage process(String queryString, QueryState queryState, QueryOptions options)
    throws RequestExecutionException, RequestValidationException
    {
        ParsedStatement.Prepared p = getStatement(queryString, queryState.getClientState());
        options.prepare(p.boundNames);
        CQLStatement prepared = p.statement;
        if (prepared.getBoundTerms() != options.getValues().size())
            throw new InvalidRequestException("Invalid amount of bind variables");

        return processStatement(prepared, queryState, options);
    }

    public static ResultMessage processStatement(CQLStatement statement,
                                                  QueryState queryState,
                                                  QueryOptions options)
    throws RequestExecutionException, RequestValidationException
    {
        logger.trace("Process {} @CL.{}", statement, options.getConsistency());
        ClientState clientState = queryState.getClientState();
        statement.checkAccess(clientState);
        statement.validate(clientState);

        ResultMessage result = statement.execute(queryState, options);
        return result == null ? new ResultMessage.Void() : result;
    }
```

接着调用UpdateStatement的父类ModificationStatement中的`execute()`，因为insert语句没有where条件，会再调用`executeWithoutCondition()`:

```java
public ResultMessage execute(QueryState queryState, QueryOptions options)
    throws RequestExecutionException, RequestValidationException
    {
        if (options.getConsistency() == null)
            throw new InvalidRequestException("Invalid empty consistency level");

        if (hasConditions() && options.getProtocolVersion() == 1)
            throw new InvalidRequestException("Conditional updates are not supported by the protocol version in use. You need to upgrade to a driver using the native protocol v2.");

        return hasConditions()
             ? executeWithCondition(queryState, options)
             : executeWithoutCondition(queryState, options);
    }


   private ResultMessage executeWithoutCondition(QueryState queryState, QueryOptions options)
    throws RequestExecutionException, RequestValidationException
    {
        ConsistencyLevel cl = options.getConsistency();
        if (isCounter())
            cl.validateCounterForWrite(cfm);
        else
            cl.validateForWrite(cfm.ksName);

        Collection<? extends IMutation> mutations = getMutations(options, false, options.getTimestamp(queryState), queryState.getSourceFrame());
        if (!mutations.isEmpty())
            StorageProxy.mutateWithTriggers(mutations, cl, false);

        return null;
    }
```

接着调用存储引擎的代理类StorageProxy中的`mutateWithTriggers()`方法：

```java
public static void mutateWithTriggers(Collection<? extends IMutation> mutations,
                                          ConsistencyLevel consistencyLevel,
                                          boolean mutateAtomically)
    throws WriteTimeoutException, UnavailableException, OverloadedException, InvalidRequestException
    {
        Collection<Mutation> augmented = TriggerExecutor.instance.execute(mutations);

        if (augmented != null)
            mutateAtomically(augmented, consistencyLevel);
        else if (mutateAtomically)
            mutateAtomically((Collection<Mutation>) mutations, consistencyLevel);
        else
            mutate(mutations, consistencyLevel);
    }
```

接入调用`mutate()`方法，这个方法中最关键的一步是调用`performWrite()`：

```java
 public static void mutate(Collection<? extends IMutation> mutations, ConsistencyLevel consistency_level)
    throws UnavailableException, OverloadedException, WriteTimeoutException
    {
 ...
 responseHandlers.add(performWrite(mutation, consistency_level, localDataCenter, standardWritePerformer, null, wt));
  ...
    }


 public static AbstractWriteResponseHandler performWrite(IMutation mutation,
                                                            ConsistencyLevel consistency_level,
                                                            String localDataCenter,
                                                            WritePerformer performer,
                                                            Runnable callback,
                                                            WriteType writeType)
    throws UnavailableException, OverloadedException
    {
        String keyspaceName = mutation.getKeyspaceName();
        AbstractReplicationStrategy rs = Keyspace.open(keyspaceName).getReplicationStrategy();

        Token tk = StorageService.getPartitioner().getToken(mutation.key());
        List<InetAddress> naturalEndpoints = StorageService.instance.getNaturalEndpoints(keyspaceName, tk);
        Collection<InetAddress> pendingEndpoints = StorageService.instance.getTokenMetadata().pendingEndpointsFor(tk, keyspaceName);

        AbstractWriteResponseHandler responseHandler = rs.getWriteResponseHandler(naturalEndpoints, pendingEndpoints, consistency_level, callback, writeType);

        // exit early if we can't fulfill the CL at this time
        responseHandler.assureSufficientLiveNodes();

        performer.apply(mutation, Iterables.concat(naturalEndpoints, pendingEndpoints), responseHandler, localDataCenter, consistency_level);
        return responseHandler;
    }
```

接着调用WritePerformer类中的内置类standardWritePerformer的apply()方法：

```java
       standardWritePerformer = new WritePerformer()
        {
            public void apply(IMutation mutation,
                              Iterable<InetAddress> targets,
                              AbstractWriteResponseHandler responseHandler,
                              String localDataCenter,
                              ConsistencyLevel consistency_level)
            throws OverloadedException
            {
                assert mutation instanceof Mutation;
                sendToHintedEndpoints((Mutation) mutation, targets, responseHandler, localDataCenter);
            }
        };
```

在`sendToHingedEndpoints()`方法中会根据host判断是否是写入本地，否则获取数据中心的信息，将消息发送到远程的其它网络节点来处理，因为这个debug是在本地测试的，因此会调用`insertLocal()`方法来执行写入：

```java
 public static void sendToHintedEndpoints(final Mutation mutation,
                                             Iterable<InetAddress> targets,
                                             AbstractWriteResponseHandler responseHandler,
                                             String localDataCenter)
    throws OverloadedException
    {
     ...

     if (destination.equals(FBUtilities.getBroadcastAddress()) && OPTIMIZE_LOCAL_REQUESTS)
      {
          insertLocal = true;
      } else
      {
                      
       String dc = DatabaseDescriptor.getEndpointSnitch().getDatacenter(destination);
                        // direct writes to local DC or old Cassandra versions
     
      ...        
      }
      ...
            if (insertLocal)
                insertLocal(mutation, responseHandler);

            if (dcGroups != null)
            {
                // for each datacenter, send the message to one node to relay the write to other replicas
                if (message == null)
                    message = mutation.createMessage();

                for (Collection<InetAddress> dcTargets : dcGroups.values())
                    sendMessagesToNonlocalDC(message, dcTargets, responseHandler);
            }
        ...
       
    }

```

在`insertLocal()`方法中调用SinkManager类的`processWriteRequest()`方法得到Mutation类，然后调用其`apply()`方法：

```java
 private static void insertLocal(final Mutation mutation, final AbstractWriteResponseHandler responseHandler)
    {
        mutation.retain();

        StageManager.getStage(Stage.MUTATION).maybeExecuteImmediately(new LocalMutationRunnable()
        {
            public void runMayThrow()
            {
                try
                {
                    IMutation processed = SinkManager.processWriteRequest(mutation);
                    if (processed != null)
                    {
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

   public void apply()
    {
        assert sourceFrame == null || sourceFrame.body.refCnt() > 0;

        Keyspace ks = Keyspace.open(keyspaceName);
        ks.apply(this, ks.metadata.durableWrites);
    }
```

调用Keyspace类的`apply()`方法，在这个方法中会写入Commit Log和Memtable，然后完成写入的过程。：

```java
    public void apply(Mutation mutation, boolean writeCommitLog)
    {
        apply(mutation, writeCommitLog, true);
    }

    public void apply(Mutation mutation, boolean writeCommitLog, boolean updateIndexes)
    {
        try (OpOrder.Group opGroup = writeOrder.start())
        {
            // write the mutation to the commitlog and memtables
            final ReplayPosition replayPosition;
            if (writeCommitLog)
            {
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
                cfs.apply(key, cf, updater, opGroup, replayPosition);
            }
        }
    }
```

其中写入Commit Log是调用`CommitLog.instance.add()`方法：

```java
    public ReplayPosition add(Mutation mutation)
    {
        Allocation alloc = add(mutation, new Allocation());
        return alloc.getReplayPosition();
    }

 private Allocation add(Mutation mutation, Allocation alloc)
    {
        assert mutation != null;

        long size = Mutation.serializer.serializedSize(mutation, MessagingService.current_version);

        long totalSize = size + ENTRY_OVERHEAD_SIZE;
        if (totalSize > MAX_MUTATION_SIZE)
        {
            throw new IllegalArgumentException(String.format("Mutation of %s bytes is too large for the maxiumum size of %s",
                                                             totalSize, MAX_MUTATION_SIZE));
        }

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
            alloc.markWritten();
        }

        executor.finishWriteFor(alloc);
        return alloc;
    }

```

接着调用`Mutation.serializer.serialize()`将内容写入输出流中：

```java
public void serialize(Mutation mutation, DataOutputPlus out, int version) throws IOException
        {
            if (version < MessagingService.VERSION_20)
                out.writeUTF(mutation.getKeyspaceName());

            ByteBufferUtil.writeWithShortLength(mutation.key(), out);

            /* serialize the modifications in the mutation */
            int size = mutation.modifications.size();
            out.writeInt(size);
            assert size > 0;
            for (Map.Entry<UUID, ColumnFamily> entry : mutation.modifications.entrySet())
                ColumnFamily.serializer.serialize(entry.getValue(), out, version);
        }
```

调用`ColumnFailySerializer.serialize()`将列信息依次写入：

```java
public void serialize(ColumnFamily cf, DataOutputPlus out, int version)
    {
        try
        {
            if (cf == null)
            {
                out.writeBoolean(false);
                return;
            }

            out.writeBoolean(true);
            serializeCfId(cf.id(), out, version);
            cf.getComparator().deletionInfoSerializer().serialize(cf.deletionInfo(), out, version);
            ColumnSerializer columnSerializer = cf.getComparator().columnSerializer();
            int count = cf.getColumnCount();
            out.writeInt(count);
            int written = 0;
            for (Cell cell : cf)
            {
                columnSerializer.serialize(cell, out);
                written++;
            }
            assert count == written: "Column family had " + count + " columns, but " + written + " written";
        }
        catch (IOException e)
        {
            throw new RuntimeException(e);
        }
    }
```

完成Commit Log的写入后，调用`ColumnFamilyStore.indexManager.updateFor()`更新索引，最后调用`ColumnFamilyStore.apply()`写入memtable。如果创建表时打开了Row Cache，也会调用`maybeUpdateRowCache()`方法更新Row Cache：

```java
public void apply(DecoratedKey key, ColumnFamily columnFamily, SecondaryIndexManager.Updater indexer,
                      OpOrder.Group opGroup, ReplayPosition replayPosition) {
        long start = System.nanoTime();

        Memtable mt = data.getMemtableFor(opGroup);
        mt.put(key, columnFamily, indexer, opGroup, replayPosition);
        maybeUpdateRowCache(key);
        metric.writeLatency.addNano(System.nanoTime() - start);
    }
```
  

