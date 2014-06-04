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

使用本地缓存后，对用户的权限完全可以控制在0.1ms以内，性能有了几个数量级的提升效果显著，但是耗时的大头还在后面等着呢。