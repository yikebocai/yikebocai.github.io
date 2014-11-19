---
layout: post
title: Tomcat源代码分析之一：导入Eclipse
categories: tech
tags: 
- tomcat
---

>Tomcat作为业界最知名的应用服务器之一，是大多数互联网公司的首选，不但是初创企业会选择它，连有些大公司像阿里也从Jboss全面转向更为轻量级的Tomcat，研究源代码可以更好地发挥它的性能和价值，在高并发压力时如果出现问题也方便定位和解决，因此准备趁这段时间有空好好研究一下。

我下载的源码版本是`7.0.50`，你也可以从[官网下载](http://tomcat.apache.org/index.html)最新版的源代码，因为我们线上使用的是这个版，因此研究的也是这个，7.0版本的总体上变化应该不大，对研究学习里面的主干内容没有什么影响。

根据[官方的文档](http://tomcat.apache.org/tomcat-7.0-doc/building.html#Building_with_Eclipse)，需要使用`ant`这个比较古老的编译工具，实在有点繁琐，网上搜了一下，[imtiger](ttp://imtiger.net/blog/2013/10/14/run-tomcat-in-idea-or-eclipse)给出了比较好的解决方案，可以自己增加pom文件，然后生成Eclipse工程，非常方便。 

## 导入Eclipse

首先将下载的`apache-tomcat-7.0.50-src.tar.gz`解压到`tomcat`目录中，然后在`tomcat`目录中创建一个`pom.xml`文件，内容如下：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>net.imtiger</groupId>
    <artifactId>tomcat-study</artifactId>
    <name>Tomcat 7.0 Study</name>
    <version>1.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>tomcat-7.0.50-src</module>
    </modules>
</project>
```

然后在`tomcat-7.0.50-src`目录下创建一个`pom.xml`文件，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">


    <modelVersion>4.0.0</modelVersion>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>Tomcat7.0</artifactId>
    <name>Tomcat7.0</name>
    <version>7.0</version>

    <build>
        <finalName>Tomcat7.0</finalName>
        <sourceDirectory>java</sourceDirectory>
        <testSourceDirectory>test</testSourceDirectory>
        <resources>
            <resource>
                <directory>java</directory>
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>test</directory>
            </testResource>
        </testResources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3</version>

                <configuration>
                    <encoding>UTF-8</encoding>
                    <source>1.6</source>
                    <target>1.6</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.4</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>ant</groupId>
            <artifactId>ant</artifactId>
            <version>1.7.0</version>
        </dependency>
        <dependency>
            <groupId>wsdl4j</groupId>
            <artifactId>wsdl4j</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>javax.xml</groupId>
            <artifactId>jaxrpc</artifactId>
            <version>1.1</version>
        </dependency>
        <dependency>
            <groupId>org.eclipse.jdt.core.compiler</groupId>
            <artifactId>ecj</artifactId>
            <version>4.2.2</version>
        </dependency>
    </dependencies>
</project>
```

最后，在`tomcat`目录下执行`mvn eclipse:eclipse`生成Eclipse工程，导入进去即可，效果如下：

![](http://yikebocai.com/myimg/20141119-tomcat-eclipse.png)