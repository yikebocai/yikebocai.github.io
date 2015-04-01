> v0.91, 2015/4/1

## 软性技能

* **不甘平庸**。每个人都有自己的梦想。
	* 向业界和身边优秀的人看齐
	* 从小事做起，并把它做好
		* 代码的逻辑正确无误
		* 代码的执行效率很高
		* 代码的结构清晰易懂
		* 代码的可扩展性良好
		* 重复的工作自动化
* **积极主动**。态度决定一切。
	* 发现工作中的问题及时反馈
	* 自己主动承担把问题解决掉
	* 协调资源推动他人一起解决
* **勤奋好学**。快速掌握一门技术是立足之本。
	* 学会翻墙
		* [曲径](https://getqujing.com)：最好的翻墙工具
		* VPN：[云梯VPN](https://www.ytvpn.com)
		* VPS：[digitalocean](https://www.digitalocean.com)、[Linode](https://www.linode.com)
		* Chrome插件：[红杏出墙](http://www.hongxingchajian.com)
	* 善用Google和Stackoverflow
	* 订阅技术文章
		* [Hacker News](https://news.ycombinator.com)：硅谷教父Paul Granham创办的YC出品
		* [Startup News](http://news.dbanotes.net)：国内IT圈的知名人士 [@Fenng](http://weibo.com/fenng) 出品
		* [ImportNew](http://www.importnew.com)：专注于Java技术分享
		* [ifeve](http://ifeve.com)：偏重于Java并发和高性能
		* [High Scalability](http://highscalability.com)：专注于大规模可扩展性系统架构
		* [码农周刊](http://weekly.manong.io)：每周推送一封，全是干货
	* 坚持读好书
	* 参加技术交流
* **保持好奇心**。知其然，也要知其所以然。
	* 阅读源码
		* [GitHub](https://github.com)
	* 尝试一门新的语言
		* Clojure
* **学会沟通**。你可以不喜欢交际，但一定要学会沟通。
	* 及时沟通，信息透明
	* 尊重他人，[学会提问](http://www.wikihow.com/Ask-a-Question-Intelligently)
		* [提问的智慧](http://www.beiww.com/doc/oss/smart-questions.html)
	* 沟通方式
		* 正式：Email
		* 非正式：微信/QQ
		* 特殊情况：面对面
	* 工作周报
		* 清晰明了美观
			* 建议使用[Markdown](http://wowubuntu.com/markdown/)工具来写
				* [马克飞象](http://maxiang.info)
				* [Cmd](https://www.zybuluo.com/mdeditor)
		* 不要敷衍写一句话周报
		* 目标、过程、结果、思考
* **学会分享**。输出是最好的输入。
	* 记笔记
		* [印象笔记](https://www.yinxiang.com)
		* [有道云笔记](http://note.youdao.com)
	* 写博客
		* [为什么要写博客](http://mindhacks.cn/2009/02/15/why-you-should-start-blogging-now/)
		* [搭建自己的独立博客](https://github.com/yikebocai/yikebocai.github.io)
	* 微博/QQ群/微信群
	* 技术会议
* **管理好自己的时间**。事情要分轻重缓急，优先做重要并紧急的事情。
	* [四象限法则](http://baike.baidu.com/link?url=6ofOPtAZzwXaFGWbxpE0m5KdOoBwlKYe6K6-tmPiOWOKFjZX7QI7mIt_F8yuXVJ8gQqJp6Yj-nY5kmdRvmu7Mq)


## 专业技能

### Linux
* 常用命令
	* 文档和目录：ls，pwd，cd，cp，mv，rm，mkdir，cat，find，tar，<，>，tail，head，more，ln，open，touch，sort，uniq
	* 权限和账户：chown，chmod，passwd，su
	* 系统和服务：ps，kill，fg，bg，nohup，reboot，shutdown，date，time，uname，df，fdisk，top，free，history，mount，chkconfig，service，crontab
	* 网络：netstat，ping，telnet，ifup，ifdown，nslookup，scp，ssh
	* 其它：alias，man，echo，xargs，grep
	* 扩展：`vim`，tree，wget，curl，yum，apt-get，brew
* 高级功能
	* awk
	* sed
	* iptable
	* [性能诊断](http://s9.51cto.com/wyfs02/M02/49/10/wKiom1QOXHDiC4XPAAFDYOVmZ7E738.jpg)
* bash
	* echo
	* if
	* for
	* 数学运算

### Web前端
* HTTP协议
	* [返回码](http://ww4.sinaimg.cn/large/5f60d6b1jw1dh2dkf1ivej.jpg)
* HTML
	* HTML5
		* WebSocket
* CSS
	* 盒模型
* JavaScript
	* ajax
* 框架
	* [bootstrap](http://getbootstrap.com)
	* [jquery](http://jquery.com)
	* [highcharts](http://www.highcharts.com)
	* echarts
	* Angularjs
* 图形
	* SVG
	* WebGL
* 工具
	* Chrome开发者模式
		* 审查元素
		* 网络请求
	* FireFox [FireBug](http://getfirebug.com) 插件
* [浏览器工作原理](http://taligarsiel.com/Projects/howbrowserswork1.htm)

### Java
* 容器类
	* List
	* Set
	* Map
* IO/NIO
	* File
	* Network
	* ByteBuffer
		* DirectByteBuffer
		* HeapByteBuffer
* 并发和多线程
	* sychronized
	* volatile
	* lock
		* ReentranLock
	* Semaphore
	* ConcurrentHashMap
	* LinkedBlockingQueue
	* Callable
	* Future
	* Executor
	* ThreadPoolExecutor
	* ForkJoinPool
* JDBC
* JVM
	* [内存模型](http://javapapers.com/wp-content/uploads/2014/10/JVM-Architecture.jpg)
		* [Heap](http://javapapers.com/wp-content/uploads/2014/10/Java-Heap-Memory.jpg)
			* 年轻代（Young Generation）
				* eden
				* S0
				* S1
			* 老年代（Old Generation，tenured）
			* 永久代（Permanent Generation）
		* Stack
		* Method Area
		* Native Method
		* PC Registers
	* 配置参数
		* -Xmx3g：设置整个堆的大小
		* -Xms3g：设置初始化堆的大小
		* -Xmn1g：设置新生代的大小
		* -XX:PermSize=192m：设置Perm区大小
		* -Xss256k：设置线程栈的大小
		* -XX:+UseConcMarkSweepGC：垃圾回收算法，CMS
		* -XX:+UseCMSInitiatingOccupancyOnly
		* -XX:CMSInitiatingOccupancyFraction=70：设置执行CMS垃圾回收的阈值
		* -XX:+PrintGCDateStamps：打印GC时间戳
		* -XX:+PrintGCDetails：打印[GC详情](https://blogs.oracle.com/poonam/entry/understanding_cms_gc_logs)
		* -Xloggc:$APP_OUTPUT/logs/gc.log：设置GC日志路径
		* -XX:+PrintGCApplicationStoppedTime
		* -XX:+PrintGCApplicationConcurrentTime
		* -XX:ErrorFile=$APP_OUTPUT/logs/hs_err_pid%p.log
	* [垃圾回收算法](http://javapapers.com/java/java-garbage-collection-introduction/)
		* Serial
		* Parallel
		* CMS
		* G1
	* javap
	* 工具
		* jps
		* jmap
		* jstack
		* jstat
* 框架
	* Webx
	* Spring
	* MyBatis
	* Netty
	* Logback
	* Drools：规则引擎
	* Druid：数据源
	* Fastjon
	* Velocity
	* Akka
* 中间件
	* Dubbo
	* Kafka
* 应用服务器
	* Tomcat
	* Jetty

### Python
* 集合
	* list/tuple
	* dict
	* set
	* 切片：lst[1:3]
	* 迭代：for c in 'abc'
	* 生成器：range(10)
* 函数
	* 函数定义
	* 字符串
		* len
		* join
		* encode
		* decode
		* 格式化
	* 高阶函数
		* map
		* reduce
		* filter
		* sorted
	* 匿名函数
	* 偏函数
* 装饰器
* 对象
	* 类和实例
	* 访问限制
	* 继承和多态
* 错误
	* try...except
* 单元测试
* 进程和线程
* 协程gevent
* 正则表达式
* 组件
	* MySQLdb
	* json
	* logging
	* datetime
	* os
	* re
* 框架
	* [flask](http://flask.pocoo.org)
* 教程
	* [廖雪峰的Python教程](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000)

### 其它语言
* Groovy
* Scala
* [Lua](http://openresty.org/download/agentzh-nginx-tutorials-zhcn.html)
* Bash

### 移动开发
* Android
* iOS

### 缓存
* 本地缓存
	* [Guava](https://github.com/google/guava) cache
	* [EHCache](http://ehcache.org)
* 分布式缓存
	* 原理
		* [一致性Hash](http://sofar.blog.51cto.com/353572/1398424)
	* 产品
		* [Memcached](http://memcached.org)
			* [spymemcached](https://github.com/couchbase/spymemcached)
		* [Redis](http://redis.io)
			* [jedis](https://github.com/xetorthio/jedis)
	* 代理
		* [Twemproxy](https://github.com/twitter/twemproxy)
		* [Codis](https://github.com/wandoulabs/codis)

### 数据库
* MySQL
	* 存储引擎
		* MyISAM
		* Innodb
	* [索引](http://tech.meituan.com/mysql-index.html)
		* Btree
		* Hash
* Berkeley DB
* [LevelDB](http://www.cnblogs.com/haippy/archive/2011/12/04/2276064.html)：Cassandra有类似的数据结构


### 代理/负载均衡
* HAProxy
* Nginx
* Apache
* Keepalive

### 大数据
* 论文
	* [BigTable](http://static.googleusercontent.com/media/research.google.com/zh-CN/us/archive/bigtable-osdi06.pdf)
	* [MapReduce](http://static.googleusercontent.com/media/research.google.com/zh-CN/us/archive/mapreduce-osdi04.pdf)
* 算法
	* DHT
	* Gossip
	* Paxos
* Hadoop
	* HDFS
	* Hive
		* Hive on Spark
		* Hive on Tez
	* Hbase
* Spark
	* SparkSQL
	* Spark Streaming
	* Spark MLLib
	* Spark Graphx
* Storm
* Cassandra
* ElasticSearch



### 运维
* 自动化
	* [ansible](http://www.ansible.com/home)：部署、配置工具
	* puppet
* 虚拟化
	* KVM / Xen
	* Docker
	* Vagent
	* OpenStack

### 测试
* TestNG
* Selenium
* Jekins

## 环境工具
* **Git**
	* git add
	* git clone
	* git commit
	* git pull
	* git push
	* git branch
	* git merge
	* git log
	* git push
	* git status
	* **gitlab**
* **Maven**
	* mvn eclipse:clean eclipse:eclipse
	* mvn clean  install
	* mvn assembly:assembly
	* mvn dependency:tree
* **Mac**
	- Alfred: Mac下第一神器
	- iTerm：比自带的终端更好用
	- zsh + oh-my-zsh: 比bash更加强大
	- tmux: 终端多窗口分屏工具
	- CatchMouse: 快捷键多屏切换
	- Reeder：最好的RSS阅读工具
	- VMWare：无缝的虚拟机软件
	- MacDown：markdown编辑器
	- Dash：各种文档资源，非常好用的snnipets
	- VirtualDiff：代码比较工具
* **Linux**
	- [Terminator](Terminator)：终端多窗口分屏工具
	- VirtualBox：开源虚拟化软件
* **Python**
	- ipython
	- pip
	- PyCharm CE
* **Java**
	* Eclipse
	* IntelliJ idea
* 其它
	* Navicat：跨平台的MySQL客户端
	* Sublime Text 2：跨平台的文本编辑器
	* Pocket：跨平台的稍后阅读工具
	* Xmind：跨平台的思维导图工具

## 推荐阅读
* Linux
	* [鸟哥的Linux私房菜](http://linux.vbird.org)：学习Linux必看
* Java
	* [Java性能优化权威指南](http://book.douban.com/subject/25828043/)：性能优化必读之作
	* [Java并发编程实战](http://book.douban.com/subject/10484692/)：深入理解Java并发
	* [深入理解Java虚拟机](http://book.douban.com/subject/24722612/)：国内为数不多介绍JVM的好书
* Python
* 数据库
	* [MySQL性能调优与架构设计](http://book.douban.com/subject/3729677/)：阿里资深DBA力作
* 大数据
	* [Apache Spark源码剖析](http://book.douban.com/subject/26340294/)：同盾大数据架构师出品
* 架构
	* [大型网站系统与Java中间件开发实践](http://book.douban.com/subject/25867042/)：来自淘宝一线架构实践
* 互联网
	* [浪潮之巅](http://book.douban.com/subject/24738302/)：了解产业发展趋势，学会顺势而为
	* [数学之美](http://book.douban.com/subject/26163454/)：了解互联网技术背后用的到数学知识
* 人文社科
	* [文明之光](http://book.douban.com/subject/26275177/)：人类如何从蒙昧一步步走向文明
* 其它
	* [影响力](http://book.douban.com/subject/1005576/)：你为什么会说“是”？
	