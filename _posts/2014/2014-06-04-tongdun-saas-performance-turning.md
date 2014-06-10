---
layout: post
title: 同盾SAAS服务性能优化总结
categories: tech
tags: 
- performance
- tongdun
---

>有大牛曾经说过，过早优化是万恶之源，深以为然。一个好的系统是在不断发展的过程中通过逐步优化进化而来，一开始就想通过完美的设计来做到高性能基本不太现实。一是业务没有发展到那一步性能再好也没用，二是创业公司各方面资源都非常紧缺没有人力和时间给技术团队去构建一个比较完美的系统，只能在业务发展的过程中一步一步地去优化完善，但这个优化完善要走在业务的前面不能拖业务的后腿才行。

我们的SAAS服务处理机制并不复杂，有一个后台管理系统，用来管理客户帐号、配置业务规则、查看业务报表等功能，本身没有太大的访问量，基本不存在性能问题。而提供API服务的系统是性能优化的重点，这个系统要为客户提供实时的服务接口调用，对性能的要求非常高，主要优化的地方也在这里。

API服务系统包括用户权限的校验、参数校验及预处理、调用规则引擎进行风险评估、保存用户事件等风个步骤。第一个版本基本是纯功能的实现，完全没有考虑性能，所有的数据都走数据库查询，使用 [siege](http://www.joedog.org/siege-home/) 进行压测之后，结果可想而知，即使在没有复杂规则的情况下，几个并发就把响应时间拖入好几秒，简直惨不忍睹，对外号称的200ms响应时间相去甚远，完全无法接受。

## 使用本地缓存

用户每次调用我们的API服务时，为了安全起见都会进行权限的校验，而每次都要查询数据库进行验证，而访问数据库都是要通过网络来进行IO交互，要知道即使使用`Hash`索引查询一次也要好几毫秒，更糟糕的是为了验证用户权限并获取相关的用户信息，还不只查一张表做一次查询，性能就更差了。其实我们的客户主要是企业客户，不像2C的网站有海量用户，数量是非常有限的，可以预期在一两年内都是一千以内，他们一但接入进来后，相关信息变化也不会频繁，完全可以缓存在内存中。

因此第一步是引入了Google的[Guava](https://code.google.com/p/guava-libraries/)框架，使用了是里面的`LoadingCache`，这个Cache有一个好处就是数据真正刷新成功之前，老的数据依然可以提供服务，新老数据可以做到无缝切换，完全避免了数据刷新时的切换问题，并且支持定时刷新、手动刷新等功能非常方便，一个基本的例子如下：

```java

public class LocalCache {

    private final Logger                      logger      = LoggerFactory.getLogger(LocalCache.class);
    

    private LoadingCache<String, DataObject> cache;
    private Integer                           refreshTime = 60;

    public void setRefreshTime(Integer refreshTime) {
        this.refreshTime = refreshTime;
    }

    public DataObject getDataObject() {
        try {
            return cache.get("dataObject");
        } catch (ExecutionException e) {
            logger.error(e.getMessage(), e);
        }

        return null;
    }

    public void init() {

        CacheLoader<String, DataObject> loader = new CacheLoader<String, DataObject>() {

            @Override
            public DataObject load(String key) {
                return loadDataObject();
            }

            @Override
            public ListenableFuture<DataObject> reload(final String key, DataObject DataObject) {
                ListenableFutureTask<DataObject> task = ListenableFutureTask.create(new Callable<DataObject>() {

                    @Override
                    public DataObject call() {
                        return loadDataObject();
                    }
                });
                ExecutorService executor = Executors.newSingleThreadExecutor();
                executor.execute(task);
                return task;

            }
        };

        RemovalListener<String, DataObject> removalListener = new RemovalListener<String, DataObject>() {

            @Override
            public void onRemoval(RemovalNotification<String, DataObject> removal) {
                logger.info("refresh local cache:" + removal.getKey());
            }

        };

        // 每小时刷新一次
        cache = CacheBuilder.newBuilder().refreshAfterWrite(refreshTime, TimeUnit.MINUTES).removalListener(removalListener).build(loader);

    }

 	private DataObject loadDataObject() {
 		// load your user info 
    }

    // 显式刷新
    public void refresh() {
        cache.refresh("dataObject");
    }
}


```

使用本地缓存后，对用户的权限完全可以控制在0.1ms以内，并且在高并发下也不会有性能的急剧下降，性能有了几个数量级的提升效果显著，但是耗时的大头还在后面等着呢。

## 结果存储异步化

前面有提到，整个风险评估结束后还需要保存用户每次请求的结果，一个是为下次做velocity计算使用，一个是为用户查询和生成统计报表用，因此原始请求内容、经过预处理后的数据、风险评估结果等信息都需要保存到数据库中。因为之前在阿里时做过这方面的优化，因此一开始就把结果的存储这一步从整个调用过程中剥离了出来，使用了异步线程来处理，为了控制高并发时线程池的数量，创建了一个线程池，设置固定的线程来异步存储数据。

但这种方式在实际压测时效果并不好，因为事件存储尤其是velocity相关的数据存储需要访问多个表，存储所需要的时间至少要好几百毫秒，并发稍微一高线程池中的线程就会被用光，导致用户调用线程被阻塞，并发仍然上不去，并且更要命的是，多线程更新同一张表的同一行数据时会加锁，导致性能急剧下降。原来是国际站时，风控系统是采用这种方式，当时只所以能够正常运行的原因是因为涉及到写数据的表比较少，插入速度相当快，因此在一定量的线程池的情况下也能满足大多数场景的要求，但也发现在高峰期时，也会出现线程池爆满的情况。

淘宝的风控系统`MTee`刚开始主要通过异步的方式接收事件进行处理，引入了本地队列的概念，数据接收进来后什么都先不处理，直接丢到本地队列，该队列使用`Berkeley DB`来实现，然后使用固定的线程在顺序处理，这样非常好地做到了削峰填谷，避免高峰期时因为系统阻塞而降低响应速度。在做异步化存储优化时也借鉴了这个思路，但根据业务特点做了扩展。异步处理时并没有在一个线程里完成所有的事件存储处理，而是根据情况不同的数据存储由不同的线程处理，每类处理器叫一个`worker`，每个worker只处理一种类型的存储，可以根据存储的优先级设定不同的线程优化级，让对时间比较敏感的Velocity存储优先处理。队列有头尾指针，每增加一个数据进去尾指针加1，而每次处理完一个数据头指针加1，每个worker对应一个头指针，当所有woker都处理完同一个数据时，才可以真正删除掉。

通过存储异步化，彻底将不影响返回风险评估结果的步骤分离出去了，但还是有一点点同步过程在里面，就是在事件往队列中添加时，需要对尾指针加锁，以避免指针和实际添加进去的数据数量不一致，但因为这个过程还是非常快的，影响不是很大。

## 使用Cache服务器

Velocity数据存储都是以KV的方式存储在数据库中，每次做velocity计算时都要从数据库中查询，开销还是很大的，50并发压测时响应时间还是会比较大，这些数据结构当初设计时就想好了要往Cache里存，因此很自然地使用了Memcached来做一层缓存，写入时先写到Memcached，读取时也先从cache中读取，命令率比较高的情况下，响应时间可以下降约5倍左右。

