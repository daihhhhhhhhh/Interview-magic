# MongoDB体系结构与底层原理

## 前言

本节将介绍MongoDB的底层结构和相关的实现原理

## MongoDB体系结构

### NoSQL 和 MongoDB

NoSQL=Not Only SQL，支持类似SQL的功能， 与Relational Database相辅相成。其性能较高，不使用SQL意味着没有结构化的存储要求(SQL为结构化的查询语句)，没有约束之后架构更加灵活。
NoSQL数据库四大家族 列存储 Hbase,键值(Key-Value)存储 Redis,图像存储 Neo4j,文档存储 MongoDB

MongoDB 是一个基于分布式文件存储的数据库，由 C++ 编写，可以为 WEB 应用提供可扩展、高性能、易部署的数据存储解决方案。

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库中功能最丰富、最像关系数据库的。在高负载的情况下，通过添加更多的节点，可以保证服务器性能。结构体系图如下：

![](https://img-blog.csdnimg.cn/20210107085915528.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)

### MongoDB 和RDBMS(关系型数据库)对比

![](https://img-blog.csdnimg.cn/20210107090009270.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)

### 什么是BSON

BSON是一种类json的一种二进制形式的存储格式，简称Binary JSON，它和JSON一样，支持内嵌的文档对象和数组对象，但是BSON有JSON没有的一些数据类型，如Date和Binary Data类型。BSON可以做为网络数据交换的一种存储形式,是一种schema-less的存储形式，它的优点是灵活性高，但它的缺点是空间利用率不是很理想。
{key:value,key2:value2} 这是一个BSON的例子，其中key是字符串类型,后面的value值，它的类型一般
是字符串,double,Array,ISODate等类型。
BSON有三个特点：轻量性、可遍历性、高效性

### BSON在MongoDB中的使用

MongoDB使用了BSON这种结构来存储数据和网络数据交换。把这种格式转化成一文档这个概念(Document)，这里的一个Document也可以理解成关系数据库中的一条记录(Record)，只是这里的Document的变化更丰富一些，如Document可以嵌套。

MongoDB中Document 中 可以出现的数据类型

![](https://img-blog.csdnimg.cn/20210107090153182.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)

至于MongoDB的基础操作使用，读者可以参考MongoDB官方文档，笔者不再赘述。

### MongoDB索引Index

索引是一种单独的、物理的对数据库表中一列或多列的值进行排序的一种存储结构，它是某个表中一列或若干列值的集合和相应的指向表中物理标识这些值的数据页的逻辑指针清单。索引的作用相当于图书的目录，可以根据目录中的页码快速找到所需的内容。索引目标是提高数据库的查询效率，没有索引的话，查询会进行全表扫描（scan every document in a collection）,数据量大时严重降低了查询效率。默认情况下Mongo在一个集合（collection）创建时，自动地对集合的_id创建了唯一索引。

#### 索引类型

单键索引 (Single Field)
MongoDB支持所有数据类型中的单个字段索引，并且可以在文档的任何字段上定义。
对于单个字段索引，索引键的排序顺序无关紧要，因为MongoDB可以在任一方向读取索引。
特殊的单键索引 过期索引 TTL （ Time To Live）
TTL索引是MongoDB中一种特殊的索引， 可以支持文档在一定时间之后自动过期删除，目前TTL索引
只能在单字段上建立，并且字段类型必须是日期类型。
复合索引(Compound Index）
通常我们需要在多个字段的基础上搜索表/集合，这是非常频繁的。 如果是这种情况，我们可能会考虑在MongoDB中制作复合索引。 复合索引支持基于多个字段的索引，这扩展了索引的概念并将它们扩展到索引中的更大域。
制作复合索引时要注意的重要事项包括：字段顺序与索引方向。
多键索引（Multikey indexes）
针对属性包含数组数据的情况，MongoDB支持针对数组中每一个element创建索引，Multikey indexes支持strings，numbers和nested documents
地理空间索引（Geospatial Index）
针对地理空间坐标数据创建索引。
2dsphere索引，用于存储和查找球面上的点
2d索引，用于存储和查找平面上的点
全文索引
MongoDB提供了针对string内容的文本查询，Text Index支持任意属性值为string或string数组元素的索引查询。注意：一个集合仅支持最多一个Text Index，中文分词不理想 推荐ES。
哈希索引 Hashed Index
针对属性的哈希值进行索引查询，当要使用Hashed index时，MongoDB能够自动的计算hash值，无需程序计算hash值。注：hash index仅支持等于查询，不支持范围查询

#### MongoDB 索引底层实现原理分析

MongoDB 是文档型的数据库，它使用BSON 格式保存数据，比关系型数据库存储更方便。比如之前关系型数据库中处理用户、订单等数据要建立对应的表，还要建立它们之间的关联关系。但是BSON就不一样了，我们可以把一条数据和这条数据对应的数据都存入一个BSON对象中,这种形式更简单，通俗易懂。MySql是关系型数据库，数据的关联性是非常强的，区间访问是常见的一种情况，底层索引组织数据使用B+树，B+树由于数据全部存储在叶子节点，并且通过指针串在一起，这样就很容易的进行区间遍历甚至全部遍历。MongoDB使用B-树，所有节点都有Data域，只要找到指索引就可以进行访问，单次查询从结构上来看要快于MySql。

B-树是一种自平衡的搜索树，形式很简单：

![](https://img-blog.csdnimg.cn/20210107091340399.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)

B-树的特点:
(1) 多路 非二叉树
(2) 每个节点 既保存数据 又保存索引
(3) 搜索时 相当于二分查找

B+树是B-树的变种

![](https://img-blog.csdnimg.cn/20210107091420676.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)

B+ 树的特点:
（1） 多路非二叉
（2） 只有叶子节点保存数据
（3） 搜索时 也相当于二分查找
（4） 增加了 相邻节点指针
从上面我们可以看出最核心的区别主要有俩，一个是数据的保存位置，一个是相邻节点的指向。就是这
俩造成了MongoDB和MySql的差别。
（1）B+树相邻接点的指针可以大大增加区间访问性，可使用在范围查询等，而B-树每个节点 key 和
data 在一起 适合随机读写 ，而区间查找效率很差。
（2）B+树更适合外部存储，也就是磁盘存储，使用B-结构的话，每次磁盘预读中的很多数据是用不上
的数据。因此，它没能利用好磁盘预读的提供的数据。由于节点内无 data 域，每个节点能索引的范围
更大更精确。
（3）注意这个区别相当重要，是基于（1）（2）的，B-树每个节点即保存数据又保存索引 树的深度
小，所以磁盘IO的次数很少，B+树只有叶子节点保存，较B树而言深度大磁盘IO多，但是区间访问比较
好。

## MongoDB架构

### MongoDB逻辑结构

![](https://img-blog.csdnimg.cn/20210107091605737.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)

MongoDB 与 MySQL 中的架构相差不多，底层都使用了可插拔的存储引擎以满足用户的不同需要。用户可以根据程序的数据特征选择不同的存储引擎,在最新版本的 MongoDB 中使用了 WiredTiger 作为默认的存储引擎，WiredTiger 提供了不同粒度的并发控制和压缩机制，能够为不同种类的应用提供了最好的性能和存储率。
在存储引擎上层的就是 MongoDB 的数据模型和查询语言了，由于 MongoDB 对数据的存储与 RDBMS有较大的差异，所以它创建了一套不同的数据模型和查询语言。

### MongoDB的数据模型

#### 描述数据模型

内嵌
内嵌的方式指的是把相关联的数据保存在同一个文档结构之中。MongoDB的文档结构允许一个字段或者一个数组内的值作为一个嵌套的文档
引用
引用方式通过存储数据引用信息来实现两个不同文档之间的关联,应用程序可以通过解析这些数据引用来访问相关数据

#### 如何选择数据模型

选择内嵌:
数据对象之间有包含关系 ,一般是数据对象之间有一对多或者一对一的关系 。
需要经常一起读取的数据。
有 map-reduce/aggregation 需求的数据放在一起，这些操作都只能操作单个 collection。
选择引用:
当内嵌数据会导致很多数据的重复，并且读性能的优势又不足于覆盖数据重复的弊端 。
需要表达比较复杂的多对多关系的时候 。
大型层次结果数据集 嵌套不要太深。

### MongoDB 存储引擎

#### 存储引擎概述

存储引擎是MongoDB的核心组件，负责管理数据如何存储在硬盘和内存上。MongoDB支持的存储引擎有MMAPv1 ,WiredTiger和InMemory。InMemory存储引擎用于将数据只存储在内存中，只将少量的元数据(meta-data)和诊断日志（Diagnostic）存储到硬盘文件中，由于不需要Disk的IO操作，就能获取所需的数据，InMemory存储引擎大幅度降低了数据查询的延迟（Latency）。从mongodb3.2开始默认的存储引擎是WiredTiger,3.2版本之前的默认存储引擎是MMAPv1，mongodb4.x版本不再支持MMAPv1存储引擎。

#### WiredTiger存储引擎优势

1.文档空间分配方式
WiredTiger使用的是BTree存储 MMAPV1 线性存储 需要Padding
2.并发级别
WiredTiger 文档级别锁 MMAPV1引擎使用表级锁
3.数据压缩
snappy (默认) 和 zlib ,相比MMAPV1(无压缩) 空间节省数倍。
4.内存使用
WiredTiger 可以指定内存的使用大小。
5.Cache使用
WT引擎使用了二阶缓存WiredTiger Cache, File System Cache来保证Disk上的数据的最终一致性。
而MMAPv1 只有journal 日志。

#### WiredTiger引擎包含的文件和作用

![](https://img-blog.csdnimg.cn/20210107092135327.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)

WiredTiger.basecfg: 存储基本配置信息，与 ConfigServer有关系
WiredTiger.lock: 定义锁操作
table*.wt: 存储各张表的数据
WiredTiger.wt: 存储table* 的元数据
WiredTiger.turtle: 存储WiredTiger.wt的元数据
journal: 存储WAL(Write Ahead Log)

#### WiredTiger存储引擎实现原理

##### 写请求

WiredTiger的写操作会默认写入 Cache ,并持久化到 WAL (Write Ahead Log)，每60s或Log文件达到2G做一次 checkpoint (当然我们也可以通过在写入时传入 j: true 的参数强制 journal 文件的同步 ，writeConcern { w: , j: , wtimeout: }) 产生快照文件。WiredTiger初始化时，恢复至最新的快照状态，然后再根据WAL恢复数据，保证数据的完整性。

![](https://img-blog.csdnimg.cn/20210107092616391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)

Cache是基于BTree的，节点是一个page，root page是根节点，internal page是中间索引节点，leafpage真正存储数据，数据以page为单位读写。WiredTiger采用Copy on write的方式管理写操作（insert、update、delete），写操作会先缓存在cache里，持久化时，写操作不会在原来的leaf page上进行，而是写入新分配的page，每次checkpoint都会产生一个新的root page。

##### checkpoint流程

1.对所有的table进行一次checkpoint，每个table的checkpoint的元数据更新至WiredTiger.wt
2.对WiredTiger.wt进行checkpoint，将该table checkpoint的元数据更新至临时文件WiredTiger.turtle.set
3.将WiredTiger.turtle.set重命名为WiredTiger.turtle。
4.上述过程如果中间失败，WiredTiger在下次连接初始化时，首先将数据恢复至最新的快照状态，然后根据WAL恢复数据，以保证存储可靠性

##### Journaling

在数据库宕机时 , 为保证 MongoDB 中数据的持久性，MongoDB 使用了 Write Ahead Logging 向磁盘上的 journal 文件预先进行写入。除了 journal 日志，MongoDB 还使用检查点（checkpoint）来保证数据的一致性，当数据库发生宕机时，我们就需要 checkpoint 和 journal 文件协作完成数据的恢复工作。

在数据文件中查找上一个检查点的标识符
在 journal 文件中查找标识符对应的记录
重做对应记录之后的全部操作

![](https://img-blog.csdnimg.cn/2021010709281654.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3o1OTEwNDU=,size_16,color_FFFFFF,t_70)