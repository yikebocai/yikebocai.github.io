---
layout: post
title: Cassandra源代码分析之二：读取
categories: tech
tags: 
- cassandra
---

> 距第一篇源代码分析文章 [Cassandra源代码学习之一：Eclipse环境准备和数据更新](http://yikebocai.com/2014/07/cassandra-sourcecode-1-env-update/) 时间有点久了，终于重新续接上了，希望后面能有时间计划的系列内容研究完毕都形成文章。

和Cassandra的交互有两种方法，一种是使用自带的`cqlsh`命令行工具，它是通过[Thrift](https://thrift.apache.org)接口和Cassandra进行通讯，它本身是用Python写的，而Cassandra是Java开发的，Thrift是一种支持跨平台的通用接口。第二种是通过不同开发语言的Driver来进行交互，比如像[Java版](https://github.com/datastax/java-driver)的，这种更适合在应用程序中的访问，它是基本`CQL3`这种二进制协议的，性能会更好，但前者使用者来说更友好。


这次的分析是基于官方的驱动来实现，因此要自己写一个客户端程序。另外，为了简化分析，采用单机模式进行分析，关键是把主要流程搞清楚即可。

## 准备
首先创建一个测试的keyspace和table：

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

插入三条数据，内容如下：

```
id | age | names
----+-----+-------------------------
 a3 |  15 | ['123', 'abc', 'hello']
 a2 |  20 |         ['john', 'tom']
 a1 |  30 |            ['01', '02']
```

引入官方的Java驱动，编写一个客户端程序，它的目的只有一个，就是按key查询一条数据出来，以便我们能方便在Server端分析读取的执行过程。

```java
import com.datastax.driver.core.Cluster;
import com.datastax.driver.core.Cluster.Builder;
import com.datastax.driver.core.Host;
import com.datastax.driver.core.Metadata;
import com.datastax.driver.core.ResultSet;
import com.datastax.driver.core.Row;
import com.datastax.driver.core.Session;
import com.datastax.driver.core.SocketOptions;

/** 
 * @author zxb 2014年12月29日 下午3:46:23
 */
public class CassandraTest {

    public static void main(String[] args) {
        Builder builder = Cluster.builder();
        builder.addContactPoint("127.0.0.1");

        // socket 链接配置
        // 为了调度时不至于很快中断，把超时时间设的长一点
        SocketOptions socketOptions = new SocketOptions().setKeepAlive(true).setConnectTimeoutMillis(5 * 10000).setReadTimeoutMillis(100000);
        builder.withSocketOptions(socketOptions);
        Cluster cluster = builder.build();
        Metadata metadata = cluster.getMetadata();
        System.out.printf("Connected to cluster: %s\n", metadata.getClusterName());
        for (Host host : metadata.getAllHosts()) {
            System.out.printf("Datatacenter: %s; Host: %s; Rack: %s\n", host.getDatacenter(), host.getAddress(),
                              host.getRack());
        }

        Session session = cluster.connect();

        ResultSet results = session.execute("SELECT * FROM forseti.mytab where id='a1'");
        for (Row row : results) {
            System.out.println(String.format("%-10s\t%-10s\t%-20s", row.getString("id"), row.getInt("age"),
                                             row.getList("names", String.class)));
        }
        cluster.close();
    }
}
```

## NativeServer的启动

Cassandra基于CQL3协议的通讯是基于Netty这个NIO框架的，从入口类`CassandraDaemon`的`start()`方法开始分析。

```java
    public void start()
    {
        String nativeFlag = System.getProperty("cassandra.start_native_transport");
        if ((nativeFlag != null && Boolean.parseBoolean(nativeFlag)) || (nativeFlag == null && DatabaseDescriptor.startNativeTransport()))
            nativeServer.start(); // 注1
        else
            logger.info("Not starting native transport as requested. Use JMX (StorageService->startNativeTransport()) or nodetool (enablebinary) to start it");

        String rpcFlag = System.getProperty("cassandra.start_rpc");
        if ((rpcFlag != null && Boolean.parseBoolean(rpcFlag)) || (rpcFlag == null && DatabaseDescriptor.startRpc()))
            thriftServer.start();
        else
            logger.info("Not starting RPC server as requested. Use JMX (StorageService->startRPCServer()) or nodetool (enablethrift) to start it");
    }
```

从`注1`的地方可以看出，在这里启动了`Server`的实例nativeServer，`nativeServer.start()`的具体实现如下：

```java
    private void run()
    {
        // Check that a SaslAuthenticator can be provided by the configured
        // IAuthenticator. If not, don't start the server.
        IAuthenticator authenticator = DatabaseDescriptor.getAuthenticator();
        if (authenticator.requireAuthentication() && !(authenticator instanceof ISaslAwareAuthenticator))
        {
            logger.error("Not starting native transport as the configured IAuthenticator is not capable of SASL authentication");
            isRunning.compareAndSet(true, false);
            return;
        }

        // 注1
        // Configure the server.
        eventExecutorGroup = new RequestThreadPoolExecutor();
        workerGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap()
                                    .group(workerGroup)
                                    .channel(NioServerSocketChannel.class)
                                    .childOption(ChannelOption.TCP_NODELAY, true)
                                    .childOption(ChannelOption.SO_KEEPALIVE, DatabaseDescriptor.getRpcKeepAlive())
                                    .childOption(ChannelOption.ALLOCATOR, CBUtil.allocator)
                                    .childOption(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, 32 * 1024)
                                    .childOption(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, 8 * 1024);

        final EncryptionOptions.ClientEncryptionOptions clientEnc = DatabaseDescriptor.getClientEncryptionOptions();
        if (clientEnc.enabled)
        {
            logger.info("Enabling encrypted CQL connections between client and server");
            bootstrap.childHandler(new SecureInitializer(this, clientEnc));
        }
        else
        {
            bootstrap.childHandler(new Initializer(this));
        }

        // Bind and start to accept incoming connections.
        logger.info("Using Netty Version: {}", Version.identify().entrySet());
        logger.info("Starting listening for CQL clients on {}...", socket);
        // 注2
        Channel channel = bootstrap.bind(socket).channel();
        connectionTracker.allChannels.add(channel);
        isRunning.set(true);
    }
```

`注1`处，主要是设置一些TCP协议的参数，`注2`将建立一个非阻塞的管道来接收请求。如果对这块不是很清楚的，可自行脑补Netty的使用。

## Netty的数据接收

数据的接收的入口方法是`org.apache.cassandra.transport.Message$Dispatcher.channelRead0()`：

```java
@Override
        public void channelRead0(ChannelHandlerContext ctx, Request request)
        {

            final Response response;
            final ServerConnection connection;

            try
            {
                assert request.connection() instanceof ServerConnection;
                connection = (ServerConnection)request.connection();
                
                // 注1
                QueryState qstate = connection.validateNewMessage(request.type, connection.getVersion(), request.getStreamId());
                qstate.setSourceFrame(request.getSourceFrame());

                logger.debug("Received: {}, v={}", request, connection.getVersion());

                // 注2
                response = request.execute(qstate);
                response.setStreamId(request.getStreamId());
                response.attach(connection);
                response.setSourceFrame(request.getSourceFrame());
                connection.applyStateTransition(request.type, response.type);
            }
            catch (Throwable ex)
            {
                request.getSourceFrame().release();
                // Don't let the exception propagate to exceptionCaught() if we can help it so that we can assign the right streamID.
                ctx.writeAndFlush(ErrorMessage.fromException(ex).setStreamId(request.getStreamId()), ctx.voidPromise());
                return;
            }

            logger.debug("Responding: {}, v={}", response, connection.getVersion());

            EventLoop loop = ctx.channel().eventLoop();
            Flusher flusher = flusherLookup.get(loop);
            if (flusher == null)
            {
                Flusher alt = flusherLookup.putIfAbsent(loop, flusher = new Flusher(loop));
                if (alt != null)
                    flusher = alt;
            }

            flusher.queued.add(new FlushItem(ctx, response));
            flusher.start();
        }

```

`注1`的地方会对接收到的请求做一些处理，`注2`的地方会真正地进行数据读取，这里和Servlet的处理有点类似，将用户请求封装成一个`Request`对象，然后执行业务逻辑，将返回结果封装成`Response`对象，最后返回结果。

但如果你在这里Debug的话，会发现这里会有一系列的系统查询之后才会真正执行我们的查询CQL，对应的Request也会不同，如下图所示：

![](http://yikebocai/myimg/20150129-cassandra-request-messages.png)


为了简单起见先忽略掉其它的查询，只关注最后一条真正的业务查询：

```
STARTUP {CQL_VERSION=3.0.0} 
REGISTER [TOPOLOGY_CHANGE, STATUS_CHANGE, SCHEMA_CHANGE] 
QUERY SELECT * FROM system.local WHERE key='local'
QUERY SELECT * FROM system.peers
QUERY SELECT * FROM system.schema_keyspaces
QUERY SELECT * FROM system.schema_columnfamilies
QUERY SELECT * FROM system.schema_columns
STARTUP {CQL_VERSION=3.0.0}
STARTUP {CQL_VERSION=3.0.0}
QUERY SELECT * FROM forseti.mytab where id='a1'
```

## 数据读取过程
因为CQL查询对象的Request子类是`QueryMessage`，因此我们跳到它的`execute()`方法来看具体的实现：

```java
    public Message.Response execute(QueryState state)
    {
        try
        {
            if (options.getPageSize() == 0)
                throw new ProtocolException("The page size cannot be 0");

            UUID tracingId = null;
            if (isTracingRequested())
            {
                tracingId = UUIDGen.getTimeUUID();
                state.prepareTracingSession(tracingId);
            }

            if (state.traceNextQuery())
            {
                state.createTracingSession();

                ImmutableMap.Builder<String, String> builder = ImmutableMap.builder();
                builder.put("query", query);
                if (options.getPageSize() > 0)
                    builder.put("page_size", Integer.toString(options.getPageSize()));

                Tracing.instance.begin("Execute CQL3 query", builder.build());
            }
            
            // 注1
            Message.Response response = state.getClientState().getCQLQueryHandler().process(query, state, options);
            if (options.skipMetadata() && response instanceof ResultMessage.Rows)
                ((ResultMessage.Rows)response).result.metadata.setSkipMetadata();

            if (tracingId != null)
                response.setTracingId(tracingId);

            return response;
        }
        catch (Exception e)
        {
            if (!((e instanceof RequestValidationException) || (e instanceof RequestExecutionException)))
                logger.error("Unexpected error during query", e);
            return ErrorMessage.fromException(e);
        }
        finally
        {
            Tracing.instance.stopSession();
        }
    }
```

这里最重要的只有`注1`那行代码，`state.getClientState().getCQLQueryHandler()`会获取一个`org.apache.cassandra.cql3.QueryProcessor`的实例，进入它的`process()`方法来看具体实现：

```java
    public ResultMessage process(String queryString, QueryState queryState, QueryOptions options)
    throws RequestExecutionException, RequestValidationException
    {
        ParsedStatement.Prepared p = getStatement(queryString, queryState.getClientState());
        options.prepare(p.boundNames);
        CQLStatement prepared = p.statement;
        if (prepared.getBoundTerms() != options.getValues().size())
            throw new InvalidRequestException("Invalid amount of bind variables");
        
        // 注1
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

        // 注2
        ResultMessage result = statement.execute(queryState, options);
        return result == null ? new ResultMessage.Void() : result;
    }

```
`注1`处的逻辑比较简单先不去管它，`注2`会执行`SelectStatement`类的`execute`方法，在这个方法中会完成数据的读取：

```java
    public ResultMessage.Rows execute(QueryState state, QueryOptions options) throws RequestExecutionException, RequestValidationException
    {
        // 注1
        ConsistencyLevel cl = options.getConsistency();
        if (cl == null)
            throw new InvalidRequestException("Invalid empty consistency level");

        cl.validateForRead(keyspace());

        // 注2
        int limit = getLimit(options);
        long now = System.currentTimeMillis();
        Pageable command = getPageableCommand(options, limit, now);

        // 注3
        int pageSize = options.getPageSize();
        // A count query will never be paged for the user, but we always page it internally to avoid OOM.
        // If we user provided a pageSize we'll use that to page internally (because why not), otherwise we use our default
        // Note that if there are some nodes in the cluster with a version less than 2.0, we can't use paging (CASSANDRA-6707).
        if (parameters.isCount && pageSize <= 0)
            pageSize = DEFAULT_COUNT_PAGE_SIZE;

        if (pageSize <= 0 || command == null || !QueryPagers.mayNeedPaging(command, pageSize))
        {
            return execute(command, options, limit, now);
        }
        else
        {
            // 注4
            QueryPager pager = QueryPagers.pager(command, cl, options.getPagingState());
            if (parameters.isCount)
                return pageCountQuery(pager, options, pageSize, now, limit);

            // We can't properly do post-query ordering if we page (see #6722)
            if (needsPostQueryOrdering())
                throw new InvalidRequestException("Cannot page queries with both ORDER BY and a IN restriction on the partition key; you must either remove the "
                                                + "ORDER BY or the IN and sort client side, or disable paging for this query");

            // 注5
            List<Row> page = pager.fetchPage(pageSize);
            // 注6
            ResultMessage.Rows msg = processResults(page, options, limit, now);

            return pager.isExhausted() ? msg : msg.withPagingState(pager.state());
        }
    }
```

`注1`处获取Cassandra的`Consistency Level`，默认是`ONE`，也就是说如果你的`replication_factor`是`3`的话，Cassandra在读到一份数据时就返回结果，这样速度是最快的，但可能会牺牲一点点数据一致性，在线上生产环境我们也是用的`ONE`。`注2`会获取结果的限制条数，默认是`Integer.MAX_VALUE`，这个值貌似没有太大作用。`注3`会获取分页信息，默认是5000条，如果结果超过这个值，该方法会多次执行。`注4`生成分页器，`注5`会真正从SSTable中读取数据，这个下面会详细分解，`注6`会对上一步返回的结果做一个封装处理。

上面`注5`会进入`AbstractQueryPager`类，具体代码如下：

```java
    public List<Row> fetchPage(int pageSize) throws RequestValidationException, RequestExecutionException
    {
        if (isExhausted())
            return Collections.emptyList();

        int currentPageSize = nextPageSize(pageSize);
        
        // 注1
        List<Row> rows = filterEmpty(queryNextPage(currentPageSize, consistencyLevel, localQuery));

        ...
       

        return rows;
    }
```

上面的代码只留了重要的内容，在`注1`会下一页的内容，进入到`SliceQueryPager`类中：

```java
   protected List<Row> queryNextPage(int pageSize, ConsistencyLevel consistencyLevel, boolean localQuery)
    throws RequestValidationException, RequestExecutionException
    {
        SliceQueryFilter filter = command.filter.withUpdatedCount(pageSize);
        if (lastReturned != null)
            filter = filter.withUpdatedStart(lastReturned, cfm.comparator);

        logger.debug("Querying next page of slice query; new filter: {}", filter);
        ReadCommand pageCmd = command.withUpdatedFilter(filter);
        
        // 注1
        return localQuery
             ? Collections.singletonList(pageCmd.getRow(Keyspace.open(command.ksName)))
             : StorageProxy.read(Collections.singletonList(pageCmd), consistencyLevel);
    }
    }
 ```
 
 因为是非本地查询模式，因此`注1`会进入到`StorageProxy.read()`方法中：
 
 ```java
     /**
     * Performs the actual reading of a row out of the StorageService, fetching
     * a specific set of column names from a given column family.
     */
    public static List<Row> read(List<ReadCommand> commands, ConsistencyLevel consistency_level)
    throws UnavailableException, IsBootstrappingException, ReadTimeoutException, InvalidRequestException
    {
        if (StorageService.instance.isBootstrapMode() && !systemKeyspaceQuery(commands))
        {
            readMetrics.unavailables.mark();
            ClientRequestMetrics.readUnavailables.inc();
            throw new IsBootstrappingException();
        }

        long start = System.nanoTime();
        List<Row> rows = null;
        try
        {
            if (consistency_level.isSerialConsistency())
            {
                // make sure any in-progress paxos writes are done (i.e., committed to a majority of replicas), before performing a quorum read
                if (commands.size() > 1)
                    throw new InvalidRequestException("SERIAL/LOCAL_SERIAL consistency may only be requested for one row at a time");
                ReadCommand command = commands.get(0);

                CFMetaData metadata = Schema.instance.getCFMetaData(command.ksName, command.cfName);
                Pair<List<InetAddress>, Integer> p = getPaxosParticipants(command.ksName, command.key, consistency_level);
                List<InetAddress> liveEndpoints = p.left;
                int requiredParticipants = p.right;

                // does the work of applying in-progress writes; throws UAE or timeout if it can't
                try
                {
                    beginAndRepairPaxos(start, command.key, metadata, liveEndpoints, requiredParticipants, consistency_level);
                }
                catch (WriteTimeoutException e)
                {
                    throw new ReadTimeoutException(consistency_level, 0, consistency_level.blockFor(Keyspace.open(command.ksName)), false);
                }

                rows = fetchRows(commands, consistency_level == ConsistencyLevel.LOCAL_SERIAL ? ConsistencyLevel.LOCAL_QUORUM : ConsistencyLevel.QUORUM);
            }
            else
            {
                // 注1
                rows = fetchRows(commands, consistency_level);
            }
        }
        catch (UnavailableException e)
        {
            readMetrics.unavailables.mark();
            ClientRequestMetrics.readUnavailables.inc();
            throw e;
        }
        catch (ReadTimeoutException e)
        {
            readMetrics.timeouts.mark();
            ClientRequestMetrics.readTimeouts.inc();
            throw e;
        }
        finally
        {
            long latency = System.nanoTime() - start;
            readMetrics.addNano(latency);
            // TODO avoid giving every command the same latency number.  Can fix this in CASSADRA-5329
            for (ReadCommand command : commands)
                Keyspace.open(command.ksName).getColumnFamilyStore(command.cfName).metric.coordinatorReadLatency.update(latency, TimeUnit.NANOSECONDS);
        }
        return rows;
    }

 ```
 
 因为CF不是SIERAL，因此进入`注1`的地方，执行`fetchRows()`方法：
 
 ```java
     private static List<Row> fetchRows(List<ReadCommand> initialCommands, ConsistencyLevel consistencyLevel)
    throws UnavailableException, ReadTimeoutException
    {
        List<Row> rows = new ArrayList<>(initialCommands.size());
        // (avoid allocating a new list in the common case of nothing-to-retry)
        List<ReadCommand> commandsToRetry = Collections.emptyList();

        do
        {
            List<ReadCommand> commands = commandsToRetry.isEmpty() ? initialCommands : commandsToRetry;
            AbstractReadExecutor[] readExecutors = new AbstractReadExecutor[commands.size()];

            if (!commandsToRetry.isEmpty())
                Tracing.trace("Retrying {} commands", commandsToRetry.size());

            // send out read requests
            for (int i = 0; i < commands.size(); i++)
            {
                ReadCommand command = commands.get(i);
                assert !command.isDigestQuery();

                // 注1
                AbstractReadExecutor exec = AbstractReadExecutor.getReadExecutor(command, consistencyLevel);
                exec.executeAsync();
                readExecutors[i] = exec;
            }

            for (AbstractReadExecutor exec : readExecutors)
                exec.maybeTryAdditionalReplicas();

            // read results and make a second pass for any digest mismatches
            List<ReadCommand> repairCommands = null;
            List<ReadCallback<ReadResponse, Row>> repairResponseHandlers = null;
            for (AbstractReadExecutor exec: readExecutors)
            {
                try
                {
                    // 注2
                    Row row = exec.get();
                    if (row != null)
                    {
                        exec.command.maybeTrim(row);
                        
                        // 注3
                        rows.add(row);
                    }

                    if (logger.isDebugEnabled())
                        logger.debug("Read: {} ms.", TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - exec.handler.start));
                }
                catch (ReadTimeoutException ex)
                {
                    int blockFor = consistencyLevel.blockFor(Keyspace.open(exec.command.getKeyspace()));
                    int responseCount = exec.handler.getReceivedCount();
                    String gotData = responseCount > 0
                                   ? exec.resolver.isDataPresent() ? " (including data)" : " (only digests)"
                                   : "";

                    if (Tracing.isTracing())
                    {
                        Tracing.trace("Timed out; received {} of {} responses{}",
                                      new Object[]{ responseCount, blockFor, gotData });
                    }
                    else if (logger.isDebugEnabled())
                    {
                        logger.debug("Read timeout; received {} of {} responses{}", responseCount, blockFor, gotData);
                    }
                    throw ex;
                }
                catch (DigestMismatchException ex)
                {
                    Tracing.trace("Digest mismatch: {}", ex);

                    ReadRepairMetrics.repairedBlocking.mark();

                    // Do a full data read to resolve the correct response (and repair node that need be)
                    RowDataResolver resolver = new RowDataResolver(exec.command.ksName, exec.command.key, exec.command.filter(), exec.command.timestamp);
                    ReadCallback<ReadResponse, Row> repairHandler = new ReadCallback<>(resolver,
                                                                                       ConsistencyLevel.ALL,
                                                                                       exec.getContactedReplicas().size(),
                                                                                       exec.command,
                                                                                       Keyspace.open(exec.command.getKeyspace()),
                                                                                       exec.handler.endpoints);

                    if (repairCommands == null)
                    {
                        repairCommands = new ArrayList<>();
                        repairResponseHandlers = new ArrayList<>();
                    }
                    repairCommands.add(exec.command);
                    repairResponseHandlers.add(repairHandler);

                    MessageOut<ReadCommand> message = exec.command.createMessage();
                    for (InetAddress endpoint : exec.getContactedReplicas())
                    {
                        Tracing.trace("Enqueuing full data read to {}", endpoint);
                        MessagingService.instance().sendRR(message, endpoint, repairHandler);
                    }
                }
            }

            commandsToRetry.clear();

            // read the results for the digest mismatch retries
            if (repairResponseHandlers != null)
            {
                for (int i = 0; i < repairCommands.size(); i++)
                {
                    ReadCommand command = repairCommands.get(i);
                    ReadCallback<ReadResponse, Row> handler = repairResponseHandlers.get(i);

                    Row row;
                    try
                    {
                        row = handler.get();
                    }
                    catch (DigestMismatchException e)
                    {
                        throw new AssertionError(e); // full data requested from each node here, no digests should be sent
                    }

                    RowDataResolver resolver = (RowDataResolver)handler.resolver;
                    try
                    {
                        // wait for the repair writes to be acknowledged, to minimize impact on any replica that's
                        // behind on writes in case the out-of-sync row is read multiple times in quick succession
                        FBUtilities.waitOnFutures(resolver.repairResults, DatabaseDescriptor.getWriteRpcTimeout());
                    }
                    catch (TimeoutException e)
                    {
                        Tracing.trace("Timed out on digest mismatch retries");
                        int blockFor = consistencyLevel.blockFor(Keyspace.open(command.getKeyspace()));
                        throw new ReadTimeoutException(consistencyLevel, blockFor-1, blockFor, true);
                    }

                    // retry any potential short reads
                    ReadCommand retryCommand = command.maybeGenerateRetryCommand(resolver, row);
                    if (retryCommand != null)
                    {
                        Tracing.trace("Issuing retry for read command");
                        if (commandsToRetry == Collections.EMPTY_LIST)
                            commandsToRetry = new ArrayList<>();
                        commandsToRetry.add(retryCommand);
                        continue;
                    }

                    if (row != null)
                    {
                        command.maybeTrim(row);
                        rows.add(row);
                    }
                }
            }
        } while (!commandsToRetry.isEmpty());

        return rows;
    }
 
 ```
 
在`注1`的地方会创建一个异步线程进行SSTable的读取，在`注2`处通过Future机制获取异步线程的执行结果，并在`注3`处把结果添加到返回列表中。异步线程的启动实际上会执行如下代码：

```java
    private static class NeverSpeculatingReadExecutor extends AbstractReadExecutor
    {
        public NeverSpeculatingReadExecutor(ReadCommand command, ConsistencyLevel consistencyLevel, List<InetAddress> targetReplicas)
        {
            super(command, consistencyLevel, targetReplicas);
        }

        public void executeAsync()
        {
            // 注1
            makeDataRequests(targetReplicas.subList(0, 1));
            if (targetReplicas.size() > 1)
                makeDigestRequests(targetReplicas.subList(1, targetReplicas.size()));
        }

        public void maybeTryAdditionalReplicas()
        {
            // no-op
        }

        public Collection<InetAddress> getContactedReplicas()
        {
            return targetReplicas;
        }
    }
    
    
        protected void makeDataRequests(Iterable<InetAddress> endpoints)
    {
        boolean readLocal = false;
        for (InetAddress endpoint : endpoints)
        {
            if (isLocalRequest(endpoint))
            {
                readLocal = true;
            }
            else
            {
                logger.trace("reading data from {}", endpoint);
                MessagingService.instance().sendRR(command.createMessage(), endpoint, handler);
            }
        }
        if (readLocal)
        {
            // 注2 
            logger.trace("reading data locally");
            StageManager.getStage(Stage.READ).maybeExecuteImmediately(new LocalReadRunnable(command, handler));
        }
    }
```

注2处`StageManager.getStage(Stage.READ)`会进入到`SEPExecutor`类：


```java
    public void maybeExecuteImmediately(Runnable command)
    {
        FutureTask<?> ft = newTaskFor(command, null);
        if (!takeWorkPermit(false))
        {   //注1
            addTask(ft);
        }
        else
        {
            try
            {
                ft.run();
            }
            finally
            {
                returnWorkPermit();
                // we have to maintain our invariant of always scheduling after any work is performed
                // in this case in particular we are not processing the rest of the queue anyway, and so
                // the work permit may go wasted if we don't immediately attempt to spawn another worker
                maybeSchedule();
            }
        }
    }
```

`注1`处会把要执行的任务放到队列中：

```java
    protected void addTask(FutureTask<?> task)
    {
        // we add to the queue first, so that when a worker takes a task permit it can be certain there is a task available
        // this permits us to schedule threads non-spuriously; it also means work is serviced fairly
        
        // 注1
        tasks.add(task);
        int taskPermits;
        while (true)
        {
            long current = permits.get();
            taskPermits = taskPermits(current);
            // because there is no difference in practical terms between the work permit being added or not (the work is already in existence)
            // we always add our permit, but block after the fact if we breached the queue limit
            if (permits.compareAndSet(current, updateTaskPermits(current, taskPermits + 1)))
                break;
        }

        if (taskPermits == 0)
        {
            // we only need to schedule a thread if there are no tasks already waiting to be processed, as
            // the original enqueue will have started a thread to service its work which will have itself
            // spawned helper workers that would have either exhausted the available tasks or are still being spawned.
            // to avoid incurring any unnecessary signalling penalties we also do not take any work to hand to the new
            // worker, we simply start a worker in a spinning state
            
            // 注2
            pool.maybeStartSpinningWorker();
        }
        ...
        
      }

```

`注1`处将任务放到队列中，`注2`处准备启动SEPWorker类，进入到`SharedExecutorPool`类中：

```java
    void maybeStartSpinningWorker()
    {
        // in general the workers manage spinningCount directly; however if it is zero, we increment it atomically
        // ourselves to avoid starting a worker unless we have to
        int current = spinningCount.get();
        if (current == 0 && spinningCount.compareAndSet(0, 1))
            // 注1
            schedule(Work.SPINNING);
    }
    
        void schedule(Work work)
    {
        // we try to hand-off our work to the spinning queue before the descheduled queue, even though we expect it to be empty
        // all we're doing here is hoping to find a worker without work to do, but it doesn't matter too much what we find;
        // we atomically set the task so even if this were a collection of all workers it would be safe, and if they are both
        // empty we schedule a new thread
        Map.Entry<Long, SEPWorker> e;
        while (null != (e = spinning.pollFirstEntry()) || null != (e = descheduled.pollFirstEntry()))
            // 注2
            if (e.getValue().assign(work, false))
                return;

        if (!work.isStop())
            new SEPWorker(workerId.incrementAndGet(), work, this);
    }
    
```

`注2`处会进入到`SEPWorker`类中：

```java
    boolean assign(Work work, boolean self)
    {
        Work state = get();
        while (state.canAssign(self))
        {
            if (!compareAndSet(state, work))
            {
                state = get();
                continue;
            }
            // if we were spinning, exit the state (decrement the count); this is valid even if we are already spinning,
            // as the assigning thread will have incremented the spinningCount
            if (state.isSpinning())
                stopSpinning();

            // if we're being descheduled, place ourselves in the descheduled collection
            if (work.isStop())
                pool.descheduled.put(workerId, this);

            // if we're currently stopped, and the new state is not a stop signal
            // (which we can immediately convert to stopped), unpark the worker
            if (state.isStopped() && (!work.isStop() || !stop()))
                // 注1
                LockSupport.unpark(thread);
            return true;
        }
        return false;
    }
```

在`注1`处启动SEPWorker线程，进行数据的读取，上面过程的整个调用栈如下。你会发现这个调用栈也是一个SEPWorker，看来Cassandra把把请求处理也封闭成了一个任务，放到`SharedExectorPool`中，由异步线程来执行：

```
Daemon Thread [SharedPool-Worker-1] (Suspended (breakpoint at line 169 in SEPWorker))	
	SEPWorker.assign(SEPWorker$Work, boolean) line: 169	
	JMXEnabledSharedExecutorPool(SharedExecutorPool).schedule(SEPWorker$Work) line: 83	
	JMXEnabledSharedExecutorPool(SharedExecutorPool).maybeStartSpinningWorker() line: 96	
	JMXEnabledSharedExecutorPool$JMXEnabledSEPExecutor(SEPExecutor).addTask(FutureTask<?>) line: 103	
	JMXEnabledSharedExecutorPool$JMXEnabledSEPExecutor(SEPExecutor).maybeExecuteImmediately(Runnable) line: 185	
	AbstractReadExecutor$NeverSpeculatingReadExecutor(AbstractReadExecutor).makeDataRequests(Iterable<InetAddress>) line: 96	
	AbstractReadExecutor$NeverSpeculatingReadExecutor.executeAsync() line: 215	
	StorageProxy.fetchRows(List<ReadCommand>, ConsistencyLevel) line: 1239	
	StorageProxy.read(List<ReadCommand>, ConsistencyLevel) line: 1180	
	SliceQueryPager.queryNextPage(int, ConsistencyLevel, boolean) line: 85	
	SliceQueryPager(AbstractQueryPager).fetchPage(int) line: 87	
	SliceQueryPager.fetchPage(int) line: 1	
	SelectStatement.execute(QueryState, QueryOptions) line: 224	
	SelectStatement.execute(QueryState, QueryOptions) line: 1	
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

## SSTABLE读取

下面的数据读取异步线程，首先进入SEPWorker的`run()`方法：

```java
    public void run()
    {
        
        ...
        
        // 注1
        task.run();
        task = null;

        ...
    }
```

在`注1`处启动一个类型为`org.apache.cassandra.concurrent.AbstractTracingAwareExecutorService$FutureTask`的任务，然后会调用`java.util.concurrent.Executors$RunnableAdapter`的实例`callable`，如`注1`处所示，之后进入`注2`处。此处应该是一个特殊的处理，无法继续Debug进入。

```java
        public void run()
        {
            try
            {
                // 注1
                result = callable.call();
            }
            catch (Throwable t)
            {
                logger.warn("Uncaught exception on thread {}: {}", Thread.currentThread(), t);
                result = t;
                failure = true;
            }
            finally
            {
                signalAll();
                onCompletion();
            }
        }
        
        
   static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            // 注2
            task.run();
            return result;
        }
    }

```

随后将进入`StorageProxy$DroppableRunnable `的`注1`和`StorageProxy$LocalReadRunnable`的`注2`处，此时才又真正回归到具体的业务逻辑处理上来，前面的异步线程处理逻辑实在有点复杂。

```java
    private static abstract class DroppableRunnable implements Runnable
    {
       
        public final void run()
        {
            try
            {
                if (TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - constructionTime) > DatabaseDescriptor.getTimeout(verb))
                {
                    MessagingService.instance().incrementDroppedMessages(verb);
                    return;
                }

                try
                {   // 注1
                    runMayThrow();
                } catch (Exception e)
                {
                    throw new RuntimeException(e);
                }
            }
            finally
            {
                cleanup();
            }
        }

    }


    static class LocalReadRunnable extends DroppableRunnable
    {
       

        protected void runMayThrow()
        {
            Keyspace keyspace = Keyspace.open(command.ksName);
            
            //注2
            Row r = command.getRow(keyspace);
            ReadResponse result = ReadVerbHandler.getResponse(command, r);
            MessagingService.instance().addLatency(FBUtilities.getBroadcastAddress(), TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start));
            handler.response(result);
        }
    }
```

上面的`command`变量是`SliceFromReadCommand`类的实例，然后直接调用`Keyspace`的`getRow()`方法，具体方法如下：

```java
    public Row getRow(Keyspace keyspace)
    {
        DecoratedKey dk = StorageService.getPartitioner().decorateKey(key);
        return keyspace.getRow(new QueryFilter(dk, cfName, filter, timestamp));
    }
```

然后在`注1`处获取`ColumnFamily`的返回对象，此时数据已读取完毕，代码如下：

```java
    public Row getRow(QueryFilter filter)
    {
        ColumnFamilyStore cfStore = getColumnFamilyStore(filter.getColumnFamilyName());
        
        //注1
        ColumnFamily columnFamily = cfStore.getColumnFamily(filter);
        return new Row(filter.key, columnFamily);
    }
```

之后进入到`ColumnFamilyStore`的方法中，从`注1`处可以看到，如果开启了`RowCache`，直接从`RowCache`中获取结果，否则走`注2`处的代码，并走到`注3`处：
```java
    public ColumnFamily getColumnFamily(QueryFilter filter) {
        assert name.equals(filter.getColumnFamilyName()) : filter.getColumnFamilyName();

        ColumnFamily result = null;

        long start = System.nanoTime();
        try {
            int gcBefore = gcBefore(filter.timestamp);
            
            // 注1
            if (isRowCacheEnabled()) {
                assert !isIndex(); // CASSANDRA-5732
                UUID cfId = metadata.cfId;

                ColumnFamily cached = getThroughCache(cfId, filter);
                if (cached == null) {
                    logger.trace("cached row is empty");
                    return null;
                }

                result = cached;
            } else {
                // 注2
                ColumnFamily cf = getTopLevelColumns(filter, gcBefore);

                if (cf == null) return null;

                result = removeDeletedCF(cf, gcBefore);
            }

            ... 
    }
    
        public ColumnFamily getTopLevelColumns(QueryFilter filter, int gcBefore) {
        Tracing.trace("Executing single-partition query on {}", name);
        CollationController controller = new CollationController(this, filter, gcBefore);
        ColumnFamily columns;
        try (OpOrder.Group op = readOrdering.start()) {
            // 注3
            columns = controller.getTopLevelColumns(Memtable.MEMORY_POOL.needToCopyOnHeap());
        }
        metric.updateSSTableIterated(controller.getSstablesIterated());
        return columns;
    }
```

下面进入到`CollationController`的`collectAllData()`方法中，如`注1`处所示。在这个方法中会对获取的数据进行合并处理，主要是`Memtable`和`SSTable`中读取到的数据的合并，并且可能会有多个`Memtable`和`SSTable`，所以从`注2`和`注3`处可以看到会有一个循环。`Memtable`在测试时并没有数据，这块会直接跳过，`SSTable`可能会有多个对应的文件，比如：

```
cassandra/cdata/data/forseti/mytab-02cde5108b5011e4a88bd19b1a65c2ff/forseti-mytab-ka-4-Data.db
cassandra/cdata/data/forseti/mytab-02cde5108b5011e4a88bd19b1a65c2ff/forseti-mytab-ka-2-Data.db
cassandra/cdata/data/forseti/mytab-02cde5108b5011e4a88bd19b1a65c2ff/forseti-mytab-ka-1-Data.db
```

在`注4`处会获取数据读取的一个迭代器，但真正从磁盘文件读取数据会在`注5`处执行：

```java
    public ColumnFamily getTopLevelColumns(boolean copyOnHeap)
    {
        return filter.filter instanceof NamesQueryFilter
               && cfs.metadata.getDefaultValidator() != CounterColumnType.instance
               ? collectTimeOrderedData(copyOnHeap)
               //注1
               : collectAllData(copyOnHeap);
    }
    
       private ColumnFamily collectAllData(boolean copyOnHeap)
    {
        Tracing.trace("Acquiring sstable references");
        ColumnFamilyStore.ViewFragment view = cfs.select(cfs.viewFilter(filter.key));
        List<Iterator<? extends OnDiskAtom>> iterators = new ArrayList<>(Iterables.size(view.memtables) + view.sstables.size());
        ColumnFamily returnCF = ArrayBackedSortedColumns.factory.create(cfs.metadata, filter.filter.isReversed());
        DeletionInfo returnDeletionInfo = returnCF.deletionInfo();
        try
        {
            Tracing.trace("Merging memtable tombstones");
            // 注2
            for (Memtable memtable : view.memtables)
            {
                final ColumnFamily cf = memtable.getColumnFamily(filter.key);
                if (cf != null)
                {
                    filter.delete(returnDeletionInfo, cf);
                    Iterator<Cell> iter = filter.getIterator(cf);
                    if (copyOnHeap)
                    {
                        iter = Iterators.transform(iter, new Function<Cell, Cell>()
                        {
                            public Cell apply(Cell cell)
                            {
                                return cell.localCopy(cf.metadata, HeapAllocator.instance);
                            }
                        });
                    }
                    iterators.add(iter);
                }
            }

            /*
             * We can't eliminate full sstables based on the timestamp of what we've already read like
             * in collectTimeOrderedData, but we still want to eliminate sstable whose maxTimestamp < mostRecentTombstone
             * we've read. We still rely on the sstable ordering by maxTimestamp since if
             *   maxTimestamp_s1 > maxTimestamp_s0,
             * we're guaranteed that s1 cannot have a row tombstone such that
             *   timestamp(tombstone) > maxTimestamp_s0
             * since we necessarily have
             *   timestamp(tombstone) <= maxTimestamp_s1
             * In other words, iterating in maxTimestamp order allow to do our mostRecentTombstone elimination
             * in one pass, and minimize the number of sstables for which we read a rowTombstone.
             */
            Collections.sort(view.sstables, SSTableReader.maxTimestampComparator);
            List<SSTableReader> skippedSSTables = null;
            long mostRecentRowTombstone = Long.MIN_VALUE;
            long minTimestamp = Long.MAX_VALUE;
            int nonIntersectingSSTables = 0;

            // 注3
            for (SSTableReader sstable : view.sstables)
            {
                minTimestamp = Math.min(minTimestamp, sstable.getMinTimestamp());
                // if we've already seen a row tombstone with a timestamp greater
                // than the most recent update to this sstable, we can skip it
                if (sstable.getMaxTimestamp() < mostRecentRowTombstone)
                    break;

                if (!filter.shouldInclude(sstable))
                {
                    nonIntersectingSSTables++;
                    // sstable contains no tombstone if maxLocalDeletionTime == Integer.MAX_VALUE, so we can safely skip those entirely
                    if (sstable.getSSTableMetadata().maxLocalDeletionTime != Integer.MAX_VALUE)
                    {
                        if (skippedSSTables == null)
                            skippedSSTables = new ArrayList<>();
                        skippedSSTables.add(sstable);
                    }
                    continue;
                }

                sstable.incrementReadCount();
                // 注4
                OnDiskAtomIterator iter = filter.getSSTableColumnIterator(sstable);
                iterators.add(iter);
                if (iter.getColumnFamily() != null)
                {
                    ColumnFamily cf = iter.getColumnFamily();
                    if (cf.isMarkedForDelete())
                        mostRecentRowTombstone = cf.deletionInfo().getTopLevelDeletion().markedForDeleteAt;

                    returnCF.delete(cf);
                    sstablesIterated++;
                }
            }

            int includedDueToTombstones = 0;
            // Check for row tombstone in the skipped sstables
            if (skippedSSTables != null)
            {
                for (SSTableReader sstable : skippedSSTables)
                {
                    if (sstable.getMaxTimestamp() <= minTimestamp)
                        continue;

                    sstable.incrementReadCount();
                    OnDiskAtomIterator iter = filter.getSSTableColumnIterator(sstable);
                    ColumnFamily cf = iter.getColumnFamily();
                    // we are only interested in row-level tombstones here, and only if markedForDeleteAt is larger than minTimestamp
                    if (cf != null && cf.deletionInfo().getTopLevelDeletion().markedForDeleteAt > minTimestamp)
                    {
                        includedDueToTombstones++;
                        iterators.add(iter);
                        returnCF.delete(cf.deletionInfo().getTopLevelDeletion());
                        sstablesIterated++;
                    }
                    else
                    {
                        FileUtils.closeQuietly(iter);
                    }
                }
            }
            if (Tracing.isTracing())
                Tracing.trace("Skipped {}/{} non-slice-intersecting sstables, included {} due to tombstones", new Object[] {nonIntersectingSSTables, view.sstables.size(), includedDueToTombstones});
            // we need to distinguish between "there is no data at all for this row" (BF will let us rebuild that efficiently)
            // and "there used to be data, but it's gone now" (we should cache the empty CF so we don't need to rebuild that slower)
            if (iterators.isEmpty())
                return null;

            // 注5
            Tracing.trace("Merging data from memtables and {} sstables", sstablesIterated);
            filter.collateOnDiskAtom(returnCF, iterators, gcBefore);

            // Caller is responsible for final removeDeletedCF.  This is important for cacheRow to work correctly:
            return returnCF;        
    }
```

下面进入到`QueryFilter`类中：

```java
    public void collateOnDiskAtom(ColumnFamily returnCF,
                                  List<? extends Iterator<? extends OnDiskAtom>> toCollate,
                                  int gcBefore)
    {
        // 注1
        collateOnDiskAtom(returnCF, toCollate, filter, gcBefore, timestamp);
    }
    
       public static void collateOnDiskAtom(ColumnFamily returnCF,
                                         List<? extends Iterator<? extends OnDiskAtom>> toCollate,
                                         IDiskAtomFilter filter,
                                         int gcBefore,
                                         long timestamp)
    {
        List<Iterator<Cell>> filteredIterators = new ArrayList<>(toCollate.size());
        for (Iterator<? extends OnDiskAtom> iter : toCollate)
            filteredIterators.add(gatherTombstones(returnCF, iter));
        collateColumns(returnCF, filteredIterators, filter, gcBefore, timestamp);
    }

    public static void collateColumns(ColumnFamily returnCF,
                                      List<? extends Iterator<Cell>> toCollate,
                                      IDiskAtomFilter filter,
                                      int gcBefore,
                                      long timestamp)
    {
        Comparator<Cell> comparator = filter.getColumnComparator(returnCF.getComparator());

        Iterator<Cell> reduced = toCollate.size() == 1
                               ? toCollate.get(0)
                               //注3
                               : MergeIterator.get(toCollate, comparator, getReducer(comparator));

        filter.collectReducedColumns(returnCF, reduced, gcBefore, timestamp);
    }

```

从上面的`注3`处进入到`MergeIterator`类，在下面的`注1`处创建一个对象，在这个对象创建过程中会完成数据的读取，这个地方不仔细分析是点有迷惑的。

```java
    public static <In, Out> IMergeIterator<In, Out> get(List<? extends Iterator<In>> sources,
                                                        Comparator<In> comparator,
                                                        Reducer<In, Out> reducer)
    {
        if (sources.size() == 1)
        {
            return reducer.trivialReduceIsTrivial()
                 ? new TrivialOneToOne<>(sources, reducer)
                 : new OneToOne<>(sources, reducer);
        }
        
        // 注1
        return new ManyToOne<>(sources, comparator, reducer);
    }
```

在下面的`注1`处
```java
        public ManyToOne(List<? extends Iterator<In>> iters, Comparator<In> comp, Reducer<In, Out> reducer)
        {
            super(iters, reducer);
            this.queue = new PriorityQueue<>(Math.max(1, iters.size()));
            for (Iterator<In> iter : iters)
            {
                Candidate<In> candidate = new Candidate<>(iter, comp);
                
                // 注1
                if (!candidate.advance())
                    // was empty
                    continue;
                this.queue.add(candidate);
            }
            this.candidates = new ArrayDeque<>(queue.size());
        }
```

使用`MergeIterator`的`advance()`方法判断迭代器是否有下一个元素，如`注1`所示：

```java
       // MergeIterator.java
       protected boolean advance()
        {
            // 注1
            if (!iter.hasNext())
                return false;
            
            item = iter.next();
            return true;
        }

```

之后进入QueryFilter的方法中，如`注1`、`注2`所示，在注2处进入`SimpleSliceReader`的`computeNext()`方法：

```java
   public static Iterator<Cell> gatherTombstones(final ColumnFamily returnCF, final Iterator<? extends OnDiskAtom> iter)
    {
        return new Iterator<Cell>()
        {
            private Cell next;

            public boolean hasNext()
            {
                if (next != null)
                    return true;
                // 注1
                getNext();
                return next != null;
            }

            public Cell next()
            {
                if (next == null)
                    getNext();

                assert next != null;
                Cell toReturn = next;
                next = null;
                return toReturn;
            }

            private void getNext()
            {
                // 注2
                while (iter.hasNext())
                {
                    OnDiskAtom atom = iter.next();

                    if (atom instanceof Cell)
                    {
                        next = (Cell)atom;
                        break;
                    }
                    else
                    {
                        returnCF.addAtom(atom);
                    }
                }
            }

            public void remove()
            {
                throw new UnsupportedOperationException();
            }
        };
    }

```

后续的执行过程如下，从`注3`处开始真正从IO输入流中读取`SSTable`文件的数据：

```java
    protected OnDiskAtom computeNext()
    {
        // 注1
        if (!atomIterator.hasNext())
            return endOfData();

        OnDiskAtom column = atomIterator.next();
        if (!finishColumn.isEmpty() && comparator.compare(column.name(), finishColumn) > 0)
            return endOfData();

        return column;
    }
    
    public abstract class AbstractCell implements Cell
{
    public static Iterator<OnDiskAtom> onDiskIterator(final DataInput in,
                                                      final ColumnSerializer.Flag flag,
                                                      final int expireBefore,
                                                      final Descriptor.Version version,
                                                      final CellNameType type)
    {
        return new AbstractIterator<OnDiskAtom>()
        {   // 注2
            protected OnDiskAtom computeNext()
            {
                OnDiskAtom atom;
                try
                {
                    // 注3
                    atom = type.onDiskAtomSerializer().deserializeFromSSTable(in, flag, expireBefore, version);
                }
                catch (IOException e)
                {
                    throw new IOError(e);
                }
                if (atom == null)
                    return endOfData();

                return atom;
            }
        };
    }
    }
```

在下面的`注1`处获取的列名，比如：`age`，在`注2`处继续做反序列化的读取：

```java
        public OnDiskAtom deserializeFromSSTable(DataInput in, ColumnSerializer.Flag flag, int expireBefore, Descriptor.Version version) throws IOException
        {
           // 注1
            Composite name = type.serializer().deserialize(in);
            if (name.isEmpty())
            {
                // SSTableWriter.END_OF_ROW
                return null;
            }

            int b = in.readUnsignedByte();
            if ((b & ColumnSerializer.RANGE_TOMBSTONE_MASK) != 0)
                return type.rangeTombstoneSerializer().deserializeBody(in, name, version);
            else
                // 注2
                return type.columnSerializer().deserializeColumnBody(in, (CellName)name, b, flag, expireBefore);
        }
```

在下面的`注1`处读取`timestamp`，这是Cassandra内置的一个属性，然后在`注2`处读取实际的值，并封装成`BufferCell`对象返回：

```java
    Cell deserializeColumnBody(DataInput in, CellName name, int mask, ColumnSerializer.Flag flag, int expireBefore) throws IOException
    {
        if ((mask & COUNTER_MASK) != 0)
        {
            long timestampOfLastDelete = in.readLong();
            long ts = in.readLong();
            ByteBuffer value = ByteBufferUtil.readWithLength(in);
            return BufferCounterCell.create(name, value, ts, timestampOfLastDelete, flag);
        }
        else if ((mask & EXPIRATION_MASK) != 0)
        {
            int ttl = in.readInt();
            int expiration = in.readInt();
            long ts = in.readLong();
            ByteBuffer value = ByteBufferUtil.readWithLength(in);
            return BufferExpiringCell.create(name, value, ts, ttl, expiration, expireBefore, flag);
        }
        else
        {
            // 注1
            long ts = in.readLong();
            // 注2
            ByteBuffer value = ByteBufferUtil.readWithLength(in);
            return (mask & COUNTER_UPDATE_MASK) != 0
                   ? new BufferCounterUpdateCell(name, value, ts)
                   : ((mask & DELETION_MASK) == 0
                      ? new BufferCell(name, value, ts)
                      : new BufferDeletedCell(name, value, ts));
        }
    }

```

这个读取过程的线程栈如下所示：

```
Daemon Thread [SharedPool-Worker-1] (Suspended)	
	ColumnSerializer.deserializeColumnBody(DataInput, CellName, int, ColumnSerializer$Flag, int) line: 136	
	OnDiskAtom$Serializer.deserializeFromSSTable(DataInput, ColumnSerializer$Flag, int, Descriptor$Version) line: 86	
	AbstractCell$1.computeNext() line: 52	
	AbstractCell$1.computeNext() line: 1	
	AbstractCell$1(AbstractIterator<T>).tryToComputeNext() line: 143	
	AbstractCell$1(AbstractIterator<T>).hasNext() line: 138	
	SimpleSliceReader.computeNext() line: 82	
	SimpleSliceReader.computeNext() line: 1	
	SimpleSliceReader(AbstractIterator<T>).tryToComputeNext() line: 143	
	SimpleSliceReader(AbstractIterator<T>).hasNext() line: 138	
	SSTableSliceIterator.hasNext() line: 82	
	QueryFilter$2.getNext() line: 172	
	QueryFilter$2.hasNext() line: 155	
	MergeIterator$Candidate<In>.advance() line: 146	
	MergeIterator$ManyToOne<In,Out>.<init>(List<Iterator<In>>, Comparator<In>, Reducer<In,Out>) line: 89	
	MergeIterator<In,Out>.get(List<Iterator<In>>, Comparator<In>, Reducer<In,Out>) line: 48	
	QueryFilter.collateColumns(ColumnFamily, List<Iterator<Cell>>, IDiskAtomFilter, int, long) line: 105	
	QueryFilter.collateOnDiskAtom(ColumnFamily, List<Iterator<OnDiskAtom>>, IDiskAtomFilter, int, long) line: 81	
	QueryFilter.collateOnDiskAtom(ColumnFamily, List<Iterator<OnDiskAtom>>, int) line: 69	
	CollationController.collectAllData(boolean) line: 311	
	CollationController.getTopLevelColumns(boolean) line: 63	
	ColumnFamilyStore.getTopLevelColumns(QueryFilter, int) line: 1689	
	ColumnFamilyStore.getColumnFamily(QueryFilter) line: 1535	
	Keyspace.getRow(QueryFilter) line: 341	
	SliceFromReadCommand.getRow(Keyspace) line: 59	
	StorageProxy$LocalReadRunnable.runMayThrow() line: 1387	
	StorageProxy$LocalReadRunnable(StorageProxy$DroppableRunnable).run() line: 2054	
	Executors$RunnableAdapter<T>.call() line: 471	
	AbstractTracingAwareExecutorService$FutureTask<T>.run() line: 162	
	SEPWorker.run() line: 103	
	Thread.run() line: 744	
```

## 小结

从上面的过程来看，整个读取过程还是很复杂的，这还没有考虑其它复杂情况，比如没有where条件的情况、结果集超过1万条的情况、Memtable有数据的情况，也没有具体分析Bloom Filter、Key Cache、Row Cache的读取过程，更没有考虑跨节点时的读取情况、Read Repair的过程等，要想完全弄清楚还得花更多的时间。
