---
layout: post
title: Cassandra源代码学习之一：Eclipse环境准备
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


