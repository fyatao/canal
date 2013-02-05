<div class="blog_content">
    <div class="iteye-blog-content-contain">
<p style="font-size: 14px;">  </p>
<h1>背景</h1>
<p style="font-size: 14px;">   早期，阿里巴巴B2B公司因为存在杭州和美国双机房部署，存在跨机房同步的业务需求。不过早期的数据库同步业务，主要是基于trigger的方式获取增量变更，不过从2010年开始，阿里系公司开始逐步的尝试基于数据库的日志解析，获取增量变更进行同步，由此衍生出了增量订阅&amp;消费的业务，从此开启了一段新纪元。ps. 目前内部使用的同步，已经支持mysql5.x和oracle部分版本的日志解析</p>
<p style="font-size: 14px;"> </p>
<p style="font-size: 14px;">基于日志增量订阅&amp;消费支持的业务：</p>
<ol style="font-size: 14px;">
<li>数据库镜像</li>
<li>数据库实时备份</li>
<li>多级索引 (卖家和买家各自分库索引)</li>
<li>search build</li>
<li>业务cache刷新</li>
<li>价格变化等重要业务消息</li>
</ol>
<h1>项目介绍</h1>
<p style="font-size: 14px;">   名称：canal [kə'næl]</p>
<p style="font-size: 14px;">   译意： 水道/管道/沟渠 </p>
<p style="font-size: 14px;">   语言： 纯java开发</p>
<p style="font-size: 14px;">   定位： 基于数据库增量日志解析，提供增量数据订阅&amp;消费，目前主要支持了mysql</p>
<p style="font-size: 14px;"> </p>
<h2>工作原理</h2>
<h3 style="font-size: 14px;">mysql主备复制实现</h3>
<p><img src="http://dl.iteye.com/upload/attachment/0080/3086/468c1a14-e7ad-3290-9d3d-44ac501a7227.jpg" alt=""><br> 从上层来看，复制分成三步：</p>
<ol>
<li>master将改变记录到二进制日志(binary log)中（这些记录叫做二进制日志事件，binary log events，可以通过show binlog events进行查看）；</li>
<li>slave将master的binary log events拷贝到它的中继日志(relay log)；</li>
<li>slave重做中继日志中的事件，将改变反映它自己的数据。</li>
</ol>
<h3>canal的工作原理：</h3>
<p><img width="590" src="http://dl.iteye.com/upload/attachment/0080/3107/c87b67ba-394c-3086-9577-9db05be04c95.jpg" alt="" height="273"></p>
<p>原理相对比较简单：</p>
<ol>
<li>canal模拟mysql slave的交互协议，伪装自己为mysql slave，向mysql master发送dump协议</li>
<li>mysql master收到dump请求，开始推送binary log给slave(也就是canal)</li>
<li>canal解析binary log对象(原始为byte流)</li>
</ol>
<h1>架构</h1>
<p><img width="548" src="http://dl.iteye.com/upload/attachment/0080/3126/49550085-0cd2-32fa-86a6-f676db5b597b.jpg" alt="" height="238" style="line-height: 1.5;"></p>
<p style="color: #333333; background-image: none; margin-top: 10px; margin-bottom: 10px; font-family: Arial, Helvetica, FreeSans, sans-serif;">说明：</p>
<ul style="line-height: 1.5; color: #333333; font-family: Arial, Helvetica, FreeSans, sans-serif;">
<li>server代表一个canal运行实例，对应于一个jvm</li>
<li>instance对应于一个数据队列  （1个server对应1..n个instance)</li>
</ul>
<p>instance模块：</p>
<ul style="line-height: 1.5; color: #333333; font-family: Arial, Helvetica, FreeSans, sans-serif;">
<li>eventParser (数据源接入，模拟slave协议和master进行交互，协议解析)</li>
<li>eventSink (Parser和Store链接器，进行数据过滤，加工，分发的工作)</li>
<li>eventStore (数据存储)</li>
<li>metaManager (增量订阅&amp;消费信息管理器)</li>
</ul>

<h1>QuickStart</h1>
<p>1.  下载canal</p>
<p>下载部署包</p>
<pre name="code" class="java">wget http://canal4mysql.googlecode.com/files/canal.deployer-1.0.0.tar.gz</pre>
<p>or </p>
<p>自己编译 </p>
<pre name="code" class="java">git clone git@github.com:otter-projects/canal.git
cd canal; 
mvn clean install -Dmaven.test.skip -Denv=release</pre>
<p>    编译完成后，会在根目录下产生target/canal.deployer-$version.tar.gz </p>
<p> </p>
<p>2.  解压缩</p>
<pre name="code" class="java">mkdir /tmp/canal
tar zxvf canal.deployer-1.0.0.tar.gz  -C /tmp/canal</pre>
<p>   </p>
<p>   解压完成后，进入/tmp/canal目录，可以看到如下结构：</p>
<p> </p>
<pre name="code" class="java">drwxr-xr-x 2 jianghang jianghang  136 2013-02-05 21:51 bin
drwxr-xr-x 4 jianghang jianghang  160 2013-02-05 21:51 conf
drwxr-xr-x 2 jianghang jianghang 1.3K 2013-02-05 21:51 lib
drwxr-xr-x 2 jianghang jianghang   48 2013-02-05 21:29 logs</pre>
<p> </p>
<p>3.  配置修改</p>
<p> </p>
<p>公用参数：   </p>
<pre name="code" class="shell">vi conf/canal.properties</pre>
<pre name="code" class="java">#################################################
#########               common argument         ############# 
#################################################
canal.id= 1
canal.address=
canal.port= 11111
canal.zkServers=
# flush data to zk
canal.zookeeper.flush.period = 1000
## memory store RingBuffer size, should be Math.pow(2,n)
canal.instance.memory.buffer.size = 32768

## detecing config
canal.instance.detecting.enable = false
canal.instance.detecting.sql = insert into retl.xdual values(1,now()) on duplicate key update x=now()
canal.instance.detecting.interval.time = 3 
canal.instance.detecting.retry.threshold = 3 
canal.instance.detecting.heartbeatHaEnable = false

# support maximum transaction size, more than the size of the transaction will be cut into multiple transactions delivery
canal.instance.transactionn.size =  1024

# network config
canal.instance.network.receiveBufferSize = 16384
canal.instance.network.sendBufferSize = 16384
canal.instance.network.soTimeout = 30

#################################################
#########               destinations            ############# 
#################################################
canal.destinations= example

canal.instance.global.mode = spring 
canal.instance.global.lazy = true  ##修改为false，代表立马启动
#canal.instance.global.manager.address = 127.0.0.1:1099
canal.instance.global.spring.xml = classpath:spring/memory-instance.xml
#canal.instance.global.spring.xml = classpath:spring/default-instance.xml</pre>
<p> </p>
<p>应用参数：</p>
<pre name="code" class="shell">vi conf/example/instance.properties</pre>
<pre name="code" class="instance.properties">#################################################
## mysql serverId
canal.instance.mysql.slaveId = 1234

# position info
canal.instance.master.address = 127.0.0.1:3306 #改成自己的数据库地址
canal.instance.master.journal.name = 
canal.instance.master.position = 
canal.instance.master.timestamp = 

#canal.instance.standby.address = 
#canal.instance.standby.journal.name =
#canal.instance.standby.position = 
#canal.instance.standby.timestamp = 

# username/password
canal.instance.dbUsername = retl  #改成自己的数据库信息
canal.instance.dbPassword = retl  #改成自己的数据库信息
canal.instance.defaultDatabaseName =   #改成自己的数据库信息
canal.instance.connectionCharsetNumber = 33  #改成自己的数据库信息
canal.instance.connectionCharset = UTF-8  #改成自己的数据库信息

# table regex
canal.instance.filter.regex = .*\\..*

#################################################
</pre>
<p> </p>
<p> </p>
<p> 说明：</p>
<ul>
<li>canal.instance.connectionCharset 代表数据库的编码方式对应到java中的编码类型，比如UTF-8，GBK , ISO-8859-1</li>
<li>canal.instance.connectionCharsetNumber 代表数据库的编码方式对应mysql中的唯一id，详细的映射关系可查看：com.mysql.jdbc.CharsetMapping.INDEX_TO_CHARSET<br>针对常见的编码：<br>utf-8  &lt;=&gt;  33<br>gb2312 &lt;=&gt; 24<br>gbk &lt;=&gt; 28</li>
</ul>
<p>4.   准备启动</p>
<p> </p>
<pre name="code" class="java">sh bin/startup.sh</pre>
<p> </p>
<p>5.  查看日志</p>
<pre name="code" class="java">vi logs/canal/canal.log</pre>
<pre name="code" class="java">2013-02-05 22:45:27.967 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## start the canal server.
2013-02-05 22:45:28.113 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[10.1.29.120:11111]
2013-02-05 22:45:28.210 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## the canal server is running now ......</pre>
<p>     </p>
<p>    具体instance的日志：</p>
<pre name="code" class="java">vi logs/example/example.log</pre>
<pre name="code" class="java">2013-02-05 22:50:45.636 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
2013-02-05 22:50:45.641 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
2013-02-05 22:50:45.803 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example 
2013-02-05 22:50:45.810 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start successful....</pre>
<p> </p>
<p>6.  关闭</p>
<pre name="code" class="java">sh bin/stop.sh</pre>
<p> </p>
<p>it's over. </p>
</div>
<h1>ClientExample</h1>
