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

在`注1`处启动SEPWorker线程，进行数据的读取。

从上面的代码看起来，启动异步线程读取SSTable的过程非常复杂。