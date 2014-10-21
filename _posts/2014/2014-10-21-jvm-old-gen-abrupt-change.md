---
layout: post
title: JVM老年代突变问题
categories: tech
tags: 
- jvm
---

最近线上有个应用，老年代突然升高，过了一段时间又突然下降，从Zabbix监控上看到如下图所示：

![](http://yikebocai.com/myimg/20141015-api-old-gen-abrupt-change.png)

环境和JVM的配置为：
```
CentOS 6.4 64位
Jdk1.7.0_51-b13
-server -Xmx3g -Xms3g -Xmn768m -XX:PermSize=256m -Xss256k  -XX:+UseConcMarkSweepGC  -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:$APP_OUTPUT/logs/gc.log
```



从代码逻辑上分析不应该出现这样的情况，因为每次用户请求进来，大约在几十毫秒就会处理完毕，中间产生的临时对象最多也就几M不得了啦，为何会突然增加1G左右的空间？

首先Dump线上Heap，分析对象的占用情况，发现序列化时使用到的Jdk类库中的ByteArrayOutputStream占用了1G多的空间，确认了问题的存在，如下图所示：

![](http://yikebocai.com/myimg/20141021-ByteArrayOutputStream-buf-too-large.png)

因为无法显示这个对象到底被谁引用的，因此不得不借助BTrace这样神器，代码如下：

```java
import static com.sun.btrace.BTraceUtils.*;
import com.sun.btrace.annotations.*;
import java.lang.reflect.Field;
import java.lang.reflect.Array;

@BTrace
public class Trace{
   private static Field fd =field("java.io.ByteArrayOutputStream", "buf");
   private static Field fd2 =field("java.io.ByteArrayOutputStream", "count");

   @OnMethod(clazz="/.*ByteArrayOutputStream/",method="grow",location=@Location(Kind.RETURN))
   public static void begin(@Self Object obj,@ProbeMethodName String pmn,int minCapacity){
     
     byte[] bs=(byte[])get(fd,obj);
     if(bs.length  > 1000000 ){
        println("----------------");
        if(bs.length > 100000000){
          jstack();
        }
        println(strcat("minCapacity: ",str(minCapacity)));
        println(strcat("buf.length: ",str(bs.length)));
        println(strcat("count: ",str(getInt(fd2,obj))));
        println(timestamp());
     }
   }
}
```

经过比较久的等待，终于捕捉到了这样问题，打印内容如下：
```

java.io.ByteArrayOutputStream.grow(ByteArrayOutputStream.java:114)
java.io.ByteArrayOutputStream.ensureCapacity(ByteArrayOutputStream.java:93)
java.io.ByteArrayOutputStream.write(ByteArrayOutputStream.java:140)
java.io.ObjectOutputStream$BlockDataOutputStream.drain(ObjectOutputStream.java:1876)
java.io.ObjectOutputStream$BlockDataOutputStream.setBlockDataMode(ObjectOutputStream.java:1785)
java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1188)
java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:347)
java.util.HashMap.writeObject(HashMap.java:1132)
sun.reflect.GeneratedMethodAccessor274.invoke(Unknown Source)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:606)
java.io.ObjectStreamClass.invokeWriteObject(ObjectStreamClass.java:988)
java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1495)
java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1431)
java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1177)
java.io.ObjectOutputStream.defaultWriteFields(ObjectOutputStream.java:1547)
java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1508)
java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1431)
java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1177)
java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:347)
java.util.HashMap.writeObject(HashMap.java:1133)
sun.reflect.GeneratedMethodAccessor274.invoke(Unknown Source)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:606)
java.io.ObjectStreamClass.invokeWriteObject(ObjectStreamClass.java:988)
java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1495)
java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1431)
java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1177)
java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:347)
org.apache.commons.lang3.SerializationUtils.serialize(SerializationUtils.java:136)
org.apache.commons.lang3.SerializationUtils.serialize(SerializationUtils.java:161)
cn.fraudmetrix.forseti.api.storage.PersistableLocalQueue.add(PersistableLocalQueue.java:151)
cn.fraudmetrix.forseti.api.service.EventStoreServiceImpl.storeEvent(EventStoreServiceImpl.java:216)
...
cn.fraudmetrix.forseti.api.service.RiskServiceImpl.execute(RiskServiceImpl.java:180)
...
cn.fraudmetrix.forseti.api.module.screen.RiskService.execute(RiskService.java:60)
...
java.lang.Thread.run(Thread.java:744)
minCapacity: 582483970
buf.length: 1164967940
count: 582483965
14-10-17 下午5:01
```

同时，也观察到了，buf是在不断增加的：
```

minCapacity: 1137665
14-10-17 下午5:00
----------------
minCapacity: 2275330
14-10-17 下午5:00
----------------
minCapacity: 4550660
14-10-17 下午5:00
----------------
minCapacity: 9101315
14-10-17 下午5:00
----------------
minCapacity: 18202625
14-10-17 下午5:00
----------------
minCapacity: 36405250
14-10-17 下午5:00
----------------
minCapacity: 72810500
14-10-17 下午5:00
----------------
minCapacity: 145620995
14-10-17 下午5:00
----------------
minCapacity: 291241985
14-10-17 下午5:00
----------------
minCapacity: 582483970
14-10-17 下午5:01
----------------
minCapacity: 1137665
14-10-17 下午5:01
----------------
minCapacity: 2275330
14-10-17 下午5:01
----------------
minCapacity: 4550660
14-10-17 下午5:01
----------------
minCapacity: 9101315
14-10-17 下午5:01
----------------
minCapacity: 18202625
14-10-17 下午5:01
----------------
minCapacity: 36405250
14-10-17 下午5:01
----------------
minCapacity: 72810500
14-10-17 下午5:01
----------------
minCapacity: 145620995
14-10-17 下午5:01
----------------
minCapacity: 291241985
14-10-17 下午5:01
----------------
minCapacity: 582483970
14-10-17 下午5:01
```

到此为止，基本可以确定，是往本地队列放入对象做序列化导致的，但是令人纳闷的是放入的对象不可能很大，难道是有些对象比较特殊，导致的序列化bug所致？

首先想到时既然序列化时导致buf数组非常大，从源代码看到序列化后会返回buf中count个byte，因此想监控序列化后的byte数组大小，如果比较大就把原始内容打印出来看看到底是什么东西，结果发布到线上后一直无法触发，代码如下：
```java
byte[] bs = SerializationUtils.serialize(elm);
if (bs.length>10000000) {
   localQueueLog.warn("bs length: " +  bs.length);
   localQueueLog.warn(JSON.toJSONString(elm));
 }
```

换了一种方法，既然内存是突然升高的，又是序列化导致的，就做了内存的监控，但也无法触发，放大监控图仔细看Old区的变化，发现也不是直线上升一下子就提上去的，大约经历一分钟涨上去，测试代码如下：
```java
byte[] bs = SerializationUtils.serialize(elm);
if (freeMemory - Runtime.getRuntime().freeMemory() > 500 * 1024 * 1024) {
   localQueueLog.warn("bs length: " + (null == bs ? 0 : bs.length));
   localQueueLog.warn(JSON.toJSONString(elm));
 }
```

上述两种方法尝试都失败后，直接把所有序列化的对象全部记录到日志中，然后等出现突变后，把日志从线下拿下来，将成反从JSON串反序列化成对象，再调用`SerializationUtils.serialize`方法，然后通过VisualVM和BTrace来观察Old Heap区的变化，发现并不会出现像线上那样的突变问题，陷入深深的迷茫中了。。。