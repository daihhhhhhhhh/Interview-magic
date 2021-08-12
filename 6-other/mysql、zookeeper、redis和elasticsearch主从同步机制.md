# mysql、zookeeper、redis和elasticsearch主从同步机制

  当系统规模达到一定程度时，传统的单机模式往往无法满足，于是就有了分布式系统。分布式系统面临的问题是CAP问题 。CAP具体含义如下：
1、consistency：一致性，保持数据同步更新

2、availability：可用性，良好的响应性能

3、partition tolerance：分区容错性，可靠性

定理：任何分布式系统只可同时满足二点，没法三者兼顾。忠告：一般3种特性不能同时满足，而是应该取舍与折中。

一般来说，当数据分布在不同的机器（或者实体集，或者虚拟机，或者跨机房等）上，具体的好处有很多，通常可以拿来作为各种指标的总结如下：

1、数据分布

2、负载平衡

3、备份

4、高可用性

5、容错

    BASE模型是CAP牺牲强一致性、保证可用性的折中方案：

1、basically available-基本可用

分布式系统发生不可预知的故障时，允许损失部分可用性，如服务降级等等

2、soft state-弱状态

分布式系统不同节点间某个时刻数据允许存在中间状态，不同节点的数据副本之间进行同步时可能存在时延，如主从同步

3、eventually consistent-最终一致

分布式系统不同节点的所有数据副本，在经过一段时间数据同步后，最终达到一致状态，即保证最终一致性，不保证实时一致性



    我们通常接触的常见中间件，如mysql、zookeeper、redis、elasticsearch等都是基于BASE理论建立的，下面就简要分析它们保持主从同步的机制，作为读书笔记，欢迎斧正与提建议，谢谢。



一、mysql

     作为通用的开源关系型数据库，mysql在CAP方面有着较好的折中，mysql集群主要是通过binlong在主从DB上进行传递来保持同步的。slave的io线程从master读取二进制日志binlog，并在本地保存为中继日志relaylog，然后sql线程读取中继日志relaylog的内容并执行命令，从而保证slave和master数据同步。

![](https://img-blog.csdn.net/20161003124124683?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

具体步骤大致如下：

1、master验证连接

2、master为slave开启主从同步线程

3、slave二进制日志binlog的偏移位ssynch告诉master

4、master检查ssynch是否小于当前二进制日志binlog偏移位msynch

5、如果ssynch小于msynch，则通知slave来取数据

6、slave持续从master取数据，直至取完

7、当master更新时，master线程被激活，并将二进制日志推送给slave，slave io线程读取网络上的二进制日志binlog

8、slave的sql线程执行二进制日志binlog，同步数据



其中 二进制日志binlog具体传输的是sql命令还是最终数据行，视具体设置而定。从mysql5.1.12开始，支持3种模式来实现复制：

1、statement-based replication，SBR-基于SQL语句的复制

优点：binlog较少，网络传输效率高；binlog可以实时还原；主从版本可以不一样

缺点：必须串行执行；不是所有的update语句都能被复制，尤其是包含不确定操作的时候，如：load_file()、uuid()、user()、found_rows()、sysdate()(除非启动了sysdate is now 选项)；进行全表扫描的update时，需要比RBR更多的行级锁；复杂的语句在slave上执行耗资源严重

2、row-based replication，RBR-基于数据行的复制

RBR优点：可以并行执行，安全可靠；需要更少的行级锁，如insert ... select、包含 auto_increment字段的 insert等

RBR缺点：binlog文件大，网络传输效率低；master上执行update语句时，写入较多，可能导致频繁发生binlog的并发写问题

3、mixed-based replication，MBR-混合模式复制

对应binlog有三种模式：statement，row，mixed，其中在MBR模式中，SBR模式是默认的。





二、zookeeper

    zookeeper是开源的分布式应用程序协调服务，是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置文件的管理、集群管理、同步锁、leader 选举、队列管理等。zookeeper集群通过paxos协议变种zab来保持同步的。

   zookeeper的主要角色为：首领-leader，跟随者-follower，观察者-observer

leader

   leader是zookeeper集群的主节点，负责响应所有对zookeeper状态变更的请求（事务性更新和非事务性查询）

   对于exists，getData，getChildren等非事务性查询请求，zookeeper服务器直接本地处理，每个服务器的命名空间是一致的。对于create，setData，delete等事务性更新请求，需要统一转发给leader处理，leader保

证2-阶段或者3-阶段来处理请求。

follower

   follower响应非事务性查询，还可以处理leader的提议，并在leader提交该提议时在本地提交。leader和follower构成zookeeper集群的法定人数，参与新leader的选举、响应leader的提议。

observer

   observer只响应非事务性查询，observer和follower区别在于：observer不参加选举也不响应提议；observer不需要将事务持久化到磁盘，一旦observer重启，需要leader全量同步命名空间。

![](https://img-blog.csdn.net/20161003124428156?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

1、SNAP-全量同步

条件：peerLastZxid<minCommittedLog

说明：证明二者数据差异太大，follower数据过于陈旧，leader发送快照SNAP指令给follower全量同步数据，即leader将所有数据全量同步到follower

2、DIFF-增量同步

条件：minCommittedLog<=peerLastZxid<=maxCommittedLog

说明：证明二者数据差异不大，follower上有一些leader上已经提交的提议proposal未同步，此时增量提交这些提议即可

3、TRUNC-仅回滚同步

条件：peerLastZxid>minCommittedLog

说明：证明follower上有些提议proposal并未在leader上提交，follower需要回滚到zxid为minCommittedLog对应的事务操作

4、TRUNC+DIFF-回滚+增量同步

条件：minCommittedLog<=peerLastZxid<=maxCommittedLog且特殊场景leader a已经将事务truncA提交到本地事务日志中，但没有成功发起proposal协议进行投票就宕机了；然后集群中剔除原leader a重新选举出

新leader b，又提交了若干新的提议proposal，然后原leader a重新服务又加入到集群中，不管是否被选举为新leader。

说明：此时a，b都有一些对方未提交的事务，leader a需要先回滚truncA然后增量同步新leader b上的数据



三、redis

    redis是一个开源的使用c编写、支持网络、可基于内存支持可持久化的日志型、key-value数据库，提供多语言的API接口，通常用来作为分布式系统的缓存服务。redis集群通过rdb文件和aof文件来保持主从同步的。slave启动时连接到master，主动发送一条sync命令；然后master启动后台持久化进程，在后台进程执行完毕后，master将传送整个redis数据库rdb文件到slave，完成全量同步；slave服务器接收到数据库rdb文件后将其存盘并加载到内存中；此后，master继续将更新命令（增删改）以aof文件的形式有序传送给slave，slave执行aof文件里的命令，从而slave与master保持数据同步。其中，关于rdb和aof两种文件含义如下：

1、rdb持久化

     在指定的时间间隔内生成数据集的时间点快照

2、aof持久化

    记录执行更新操作命令，新命令会被追加到文件的末尾；redis对aof文件进行重写，使得aof文件不至于很大
    
     redis还可以同时使用aof持久化和rbd持久化。在这种情况下，当redis重启时，优先使用aof文件来还原数据集，因为aof文件通常比rdb文件所保存的数据集更完整。

![](https://img-blog.csdn.net/20161003131006675?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

     slave连接master后，会主动发起psync命令，slave会提供master_runid和offset，master验证master_runid和offset是否有效，其中master_runid作为master身份验证，offset是全局积压空间数据的偏移量。



1、完整重同步

   当slave的offset不在master暂存队列时，执行完整重同步。master返回 full resync master_runid offset，启动bgsave生成rdb文件，bgsave结束后，向slave传输，从而完成完整重同步

2、部分重同步

    当slave的offset存在于master暂存队列时，执行部分重同步。slave读取offset偏移之后的所有更新事务日志aof，然后slave执行对应事务



四、elasticsearch

     elasticsearch是一个基于lucene构建的开源，分布式，restfull搜索引擎，能够达到实时搜索，稳定，可靠，快速，支持通过http使用json进行数据索引。 

1、node-节点

     单个es实例，一台主机上部署es应用则称为一个节点

2、cluster-集群

    由若干节点组成

3、shard-分片

     一个索引会分成多个分片存储，分片数量在索引建立后不可更改
4、replica-副本

    副本是分片的一个拷贝，提高系统的容错性和查询性能

5、index-索引

    类比数据库的库
6、type-类型

    类比数据库的表

7、document-文档

     类比数据库的行，包含若干个field

8、field-字段

    搜索的最小单元，可通过mapping定义不同的属性
    
    es中master节点相对没有mysql、zookeeper、redis等对于整个集群那么重要，但是也是特别重要的。es的master监控集群的拓扑结构和健康状态，分发索引分片到集群节点，不同的是具体文档的索引主分片不一定在master上。


