---
layout: post
title: Tomcat源代码分析之二：启动和关闭
categories: tech
tags: 
- tomcat
---

>看了Tomcat的源代码之后发现，整个启动过和关闭过程还是挺复杂的，但经过抽丝剥茧，理顺之后大体流程也是比较清楚的。


在搭建Web应用时，一般都会用Tomcat或者Jboss或Jetty这样的容器，我们一般是不关心启动入口的，它被隐藏在非常深的地方，但研究Tomcat的源代码，首要的任务就是找到启动入口，这样才能一步步调试并顺藤摸瓜理清整个脉络。因为已经从别的地方看到，Tomcat启动的入口类是org.apache.catalina.startup.Bootstrap，入口方法是main()，所以就省去了寻找的过程，如果在研究其它开源代码不知道的话，该怎么办呢，一个个去查看显然不太现实，像Tomcat的java源文件有1792个，那得找到猴年马月呀。但借助shell命令，查找包括main方法的类，再根据文件名猜测入口类，就变得简单多了，Cassandra的源代码我就是使用这种方式。

```bash
find .|grep '\.java$'|grep -v test|grep -v examples|grep -v modules|xargs grep 'void main('
```

结果如下，把包含util关键词的排除掉，就很容易找到入口类了：

```
./java/org/apache/catalina/ha/deploy/FileMessageFactory.java:    public static void main(String[] args) throws Exception {
./java/org/apache/catalina/realm/RealmBase.java:    public static void main(String args[]) {
./java/org/apache/catalina/startup/Bootstrap.java:    public static void main(String args[]) {
./java/org/apache/catalina/startup/Tool.java:    public static void main(String args[]) {
./java/org/apache/catalina/tribes/membership/Constants.java:    public static void main(String[] args) throws Exception {
./java/org/apache/catalina/tribes/membership/McastService.java:    public static void main(String args[]) throws Exception {
./java/org/apache/catalina/util/ServerInfo.java:    public static void main(String args[]) {
./java/org/apache/jasper/compiler/SmapGenerator.java:    public static void main(String args[]) {
./java/org/apache/jasper/JspC.java:    public static void main(String arg[]) {
./java/org/apache/tomcat/util/bcel/classfile/Method.java:     * `public static void main(String[] args) throws IOException', e.g.
./java/org/apache/tomcat/util/bcel/classfile/Utility.java:     * `void main(String[])' and throws a `ClassFormatException' when the parsed
```

## 启动的总体流程

```
  1)进入到BootStrap#main()方法中
  
  2)创建一个Bootstrap实例，并调用它的init()方法进行初始化，包括setCatalinaHome()、setCatalinaBase()、initClassLoaders()，并创建一个Catalina类的实例catalinaDaemon。
  
  3)然后将该实例赋给Boostrap的实例daemon，并调用load(args)方法进行加载，它实际上会使用反射的方法，调用上面的catalinaDaemon实例的load()方法初始化Server、Service、Connector等
    - 在load()方法中调用createStartDigester()创建Digester实例，设置StandardServer作为默认的server，设置StandardThreadExecutor作为默认的Executor等信息
    - 在load()方法中加载conf/server.xml配置文件
    - 调用StandardServer#init()方法初始化
      - 调用initInernal()方法，注册MBean并初始化注册的所有服务，这里是在Ditester中定义的StandardServce#init()
        - 然后调用StandardService#initInternal()方法
          - 执行container.init()初始化StandardEngine类的容器实例
          - 执行executor.init()初始化所有Executor
          - 执行connector.init()初始化所有Connector，比如Ajp、Bio或Nio Connector
            - 调用initInternal()方法
              - 初始化协议处理器，比如Http11NioProtocal
              - 调用protocalHandler.init()初始化协议处理器（实际调用父类AbstractProtocal的init()方法）
                - 注册Mbean
                - 设置Endpoint的名称，比如http-nio-8080
                - 调用endpoint.init()方法初始化
                  - 调用父类AbstractEndpoint#init()方法
                    - 调用bind()方法，绑定地址和端口
              - 调用mapperListener.init()方法，初始化listener（只是注册Mbean）
 
  4）使用反射的方法，调用catalinaDaemon.start()启动Server
     - 调用startInternal()方法，启动所有service，这里默认是StandardService#start()
     - 调用LifecycleBase#start()方法
       - 调用startInteral()方法 
         - 调用（StandardEngine）container.start()启动Engine
           - 调用StandardEngine#startInternal()
             - 调用父类ContainerBase#startInternal()
               - 执行findChildren()方法获取子容器，比如StandardHost的实例
               - 将这些子容器用StartChild类包装成Callable的类，使用线程池启动
                 - 调用StartChild#call()方法
                   - 调用StandardHost#start()方法启动Host
                     - 调用StandardHost#startInternal()
                       - 调用getPipeline().addValve()增加ErrorReportValve
                       - 调用super.startInternal()使用父类ContainerBase的启动方法部署应用
                         - 调用setState() 方法设置Context，这个方法有有点误导人，它里面其实做了很多工作，不仅仅是设置个状态值
                           - 调用LifecyleBase#setStateInternal()
                             - 调用fireLifeCycleEvent(“start”,null)
                               - 调用LifecycleSurpport#fireLifecycleEvent()
                                 - 获取注册过的侦听器，这里是HostConfig
                                   - 调用HostConfig#lifecycleEvent()
                                     - 调用start()方法
                                       - 调用deployApps()部署应用
                                         - 调用deployWARs()部署war包
                                         - 调用deployDirectories()部署解压后的目录
                                           - 获取webapps目录下的所有目录，比如docs, examples, host-manager, manager, ROOT
                                        - 将之用实现Callable接口的DeployDirectory包装，放入到线程池中启动
                                          - 调用DeployDirectory#run()方法启动线程
                                            - 调用HostConfig#deployDirectories(ContextName,File)
                                              - 调用digester.parse(“webapps/docs/META-INF/context.xml")创建一个StandardContext实例
                                              - 调用host.addChild(context)将之增加到StandardHost实例中
                                                - 调用ContainerBase#addChildInternal()
                                                  - 调用child.start()启动StandardContext
         - 调用executor.start()启动Executor
         - 调用connector.start()启动Connector
           - 调用startInternal()
             - 调用protocalHandler.start()启动处理器
               - 调用父类AbstractProtocal#start()
                 - 调用Endpoint的子类，比如JIoEndpoint的start()方法
                   - 调用createExecutor()方法创建Executor
                     - 创建TaskQueue
                     - 创建一个ThreadPoolExecutor
                   - 调用startAcceptorThreads()方法启动接收器线程
                     - 调用getAcceptorThreadCount()获取要启动的接收器线程个数
                     - 调用createAcceptor()创建接收器，比如NioEndpoint.Acceptor
                     - 将它放到线程中启动（实际调用具体的NioEndpoint.Acceptor.run()）
                       - 调用countUpOrAwaitConnection()检查最大连接数，如果超过就等待
                       - 调用(ServerSocketChannel)serverSock.accept()等待下一个连接进来，此时会阻塞到这里
             - 调用mapperListner.start()启动listener

  5) 最后在控制台打印启动时间，完成Tomcat的全部启动过程
```

## 关闭的整体流程

```
1）调用（Catalina）catalinaDaemon.start()启动Tomcat之后，在方法的最后将运行到await()方法这里，并阻塞
    - 调用 getServer().await()方法，实际上是StandardServer#await()
      - 创建一个ServerSocket，侦听默认的8005端口
      - 调用serverSocket.accept()方法阻塞，等待连接的进入
      - 有连接进来后，读取请求命令是否等于SHUTDOWN，如果是的话就关闭Socket连接
2）调用（Catalina）catalinaDaemon.stop()方法，停止Tomcat
      - 获取StandardServer实例
      - 调用它的stop()方法，实际调用stopInternal()
        - 调用service.stop()停止所有服务
          - 调用StandardService.stopInternal()方法
            - 调用Connector#pause()
              - 调用(Http11NioProtocal)protocalHandler.pause(),这里会调用父类AbstractProtocal的pause()方法
                - 调用(NioEndpoint)endpoint.pause()
                  - 调用unlockAccept()
                    - 在这里结束掉Acceptor，停止接收新的连接请求
      - 调用它的destroy()方法
        - 调用StandardServer#destoryInternal()
          - 调用StandandService#destroy()方法
          - 注销MBean
```