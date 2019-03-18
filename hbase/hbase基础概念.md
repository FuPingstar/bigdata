## 分布式数据库HBase

BigTable:（起源：） __解决谷歌公司内部大规模网页搜索问题__ <br>
	`1. 建立整个网页的索引`

	设计一个网页爬虫

	在BigTable上运行MapReduce

	`2. 搜索互联网网页`

	Hbase是BagTable的开源实现

	Hbase是分布式数据库，最主要的特长是用来存储非结构化和半结构化的松散数据
	（底层文件系统是存储完全非结构化的数据）

为什么要设计HBase？

<p style="text-indent:2em">虽然已经有了HDFS和MapReduce，但是Hadoop主要解决大规模数据离线批量处理，Hadoop是没有办法满足大数据实时处理需求，随着这些年数据的大规模爆炸式增长，传统关系型数据库的扩展能力非常有限</p>

- hbase与传统数据库的区别	
    1. 数据类型：
		传统的关系型数据库用的是非常经典的关系数据模型，HBase采用了更加简单的数据模型，把数据存储为未经解释的字符串
    2. 数据操作方面：
		关系数据库中定了非常多的数据操作，增删改查，其中还设计多表的复杂连接查询，HBase不会存在复杂的表与表之间的关系，只有简单的插入，查询，删除，清空等
    3. 存储模式：
		关系数据库基于行模式进行存储，元组或者行会被连续的存储在磁盘也中，在读取数据时，需要顺序扫描每个元组，基于行模式的存储会浪费很多磁盘空间和内存带宽，Hbase是基于列进行存储，每个列族有几个文件保存，不同的列族的文件是分离的，优点是：可以降低IO开销，支持大量并发用户查询，因为仅需处理可以回答这些查询的列
    4. 数据索引：
		关系数据库可以直接针对各个不同的列，构建非常复杂的索引，HBase只有一个索引：行键，		
    5. 数据维护：
		在关系数据库做一些数据更新操作的时候，实际上里面旧的值会被新的值覆盖掉，HBase执行更新操作时，并不会删除旧的版本，而是生成一个新的版本，旧的版本仍有保留
    6. 可伸缩性：
		关系数据库很难实现水平扩展的，最多可以实现纵向扩展，HBase就是为了实现灵活的水平扩展而开发的
	
	
+ HBase访问接口：
	1. 提供了原生Java API
			shell命令 Thrift Gateway方式 REST Gateway
	2. 提供SQL类型接口
		+ pig：使用Pig Latin流式编程语言来出来Hbase中的数据
        + 数据仓库产品Hive
    3. Rest Gateway
        <p style="text-indent:2em">解除了语言限制</p>
    4. Thrift GateWay        
    5. HBase Shell


+ 数据模型
	1. Hbase是一个稀疏的多维度的排序的映射表
		+ 列限定符：
		+ 行键：
		+ 时间戳：
		+ 列族：
		    
    2. 数据坐标的概念：
		<p style="text-indent:2em">Hbase对数据的定位：采用四维坐标来定位的，必须确定行键，列族，列限定符号，时间戳，</p> 
	3. 概念试图：HBase是稀疏表，很多单元格是没有数据的 
	4. 物理视图
	5. 面向行的存储的优点和缺陷：
		+ 优点：对于传统的事物型操作，需要每次插入一条数据，会对整条记录的各项信息存入数据库
		- 缺点：对数据分析有缺陷： 
		面向列进行存储
		+ 好处：按一个列进行存储，可以带来很高的数据压缩率
	Hbase为什么要保留旧的版本呢？
		HBase大多是底层用HDFS进行存储数据的，HBase是建立在HDFS之上的，HDFS不允许对某个文件进行修改，只能追加，所以HBase无法对某个数据进行修改，只能生成新的数据		
	

+ <font color="red">实现原理</font>
    1. HBase的功能组件：	
	    + 库函数：一般用于链接每个客户端
		+ Master服务器：充当管家
		+ Region服务器：负责存储不同的Region
					
	2. HBase设计了三层结构实现Region的寻址和定位
		+ Zookeeper文件：记录了-ROOT-表的位置信息
		+ -ROOT-表：记录了.META表的Region的位置信息，—ROOT-表只能有一个region.通过-ROOT-表，可以访问到.meta表中的数据
		+ META表：记录了用户数据表的Region位置信息，.META表可以有多个region，保存了Hbase中所有用户数据表的Region位置信息
		+ 为了加速寻址，客户端会缓存位置信息，同时，需要解决缓存失效问题
	3. 运行机制
		+ 客户端
            <p style="text-indent:2em">访问HBase的接口（为了加快访问速度，客户端会在自己的缓存中维护已经访问过的Region服务器的位置信息）</p>
		+ Zookeeper服务器
            <p style="text-indent:2em">实现协同管理服务，被大量用于分布式系统，提供配置维护，域名服务，分布式同步服务，在HBase中提供管家的功能，维护和管理整个HBase集群</p>
		+ Master（主服务器):负责HBase当中表和Region的管理工作
			1. 对表增删改查
			2. 负责不同region服务器的负载均衡
			3. 负责调整分裂、合并后Region的分布
			4. 负责重新分配故障、失效的Region服务器
		+ Region服务器
		
+ <font color="red">用户读写数据过程</font>
	1. 写：
	<p style="text-indent:2em">用户写入数据，去被分配到Region服务器上去执行，先写到缓存中而不是磁盘中，先写到MemStore,为了保存数据的安全和可恢复性，必须先写到日志到Hlog中，只有当Hlog中的数据完整的被写入到磁盘当中去，才允许返回给客户端</p> 
	2. 读：
		<p style="text-indent:2em">用户读取数据的时候，Region服务器先去memstore去找，因为最新的数据都在memstore中，如果memestore找不到，再去磁盘的storefile文件中去找</p>
	3. 缓存的刷新<br>
	    <p style="text-indent:2em">系统会周期性的把MemStore缓存里的内容刷写到磁盘的StoreFile文件中，清空缓存，并在Hlog里面写入一个标记。</p>
		<p style="text-indent:2em">每次刷新都会生成一个新的StoreFile文件，因此，每个Store包含多个StoreFile文件</p>
		<p style="text-indent:2em">每个Region服务器都有一个自己的Hlog文件，每次启动都检查该文件，确认最近一次执行缓存刷新操作之后是否发生新的写入操作，如果发现更新，则先写入MemStore，再刷写到StoreFile，最后删除旧的Hlog文件，开始为用户提供服务</p>			
    4. StoreFile的合并：
		<p style="text-indent:2em">当StoreFile的大小达到一个阈值，storeFile进行合并，得到一个大的StoreFile，当大的storeFile一定大的时候，Region就开始分裂</p>
	5. HLog的工作原理
		<p style="text-indent:2em">当Region服务器发生故障，MemStore缓存中还没有写入到磁盘的文件将会全部丢失，因此，HBase采用HLog来保证系统发生故障时能恢复到正确的状态</p>
		<p style="text-indent:2em">HLog是一种预写式日志，也就是说，用户更新数据首先被写入HLog才能写入MemStore缓存中，并且知道MemStore缓存内容对应的日志已经被写入到磁盘之后，该缓存才会被刷新写入磁盘</p>
		<p style="text-indent:2em">Hlog用来恢复数据的日志文件，一个Region服务器只有一个Hlog，当Zookeeper检测到发生故障后，通知Master，Master把故障服务器上的Hlog拿过来，进行拆分，分配给相应的Region，放置在重新分配的region服务器上，Region服务器领取到分配给自己的Region对象以及与之相关的HLOg日志记录，会重新做一遍日志记录的各种记录，把日志记录中的数据写入到MemStore缓存，然后刷新到磁盘的StoreFile中， 进行数据恢复.</p>

