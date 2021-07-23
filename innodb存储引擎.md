# innodb存储引擎

数据库和实例

------

 

数据库(database):物理操作系统文件或其他形式文件类型的集合

实例(instance):mysql数据库由后台线程以及一个共享内存区组成。

 

通常情况下，两者是一对一关系；但是，在集群情况下可能存在一个数据库被多个数据实例使用的情况。

 

mysql实例在系统上的表现就是一个进程；

 

InnoDB存储架构

 

![img]()

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201614154-1247891255.png)

 

innodb 在内存中的缓存池 buffer pool ；

 

 

**innodb相关的磁盘文件**

![img]()

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201627122-2053526887.png)

 

innodb系统表空间文件：

ibdata1存放：

- 回滚段
- 所有innodb表元数据信息（这就是为什么innodb无法像myisam表一样，直接将表定义文件  表名.frm 和表数据文件 表名.ibd 拷贝到 另一个库中，因为还有部分元数据信息在ibdata1文件中）
- double write，insert buffer dump 等等

自动扩展机制

 

 

**基本参数**

 

查看innodb的配置参数

```
mysql> show global variables like "%innodb%" ；
```

基本参数：

innodb_data_home_dir:        系统表空间文件ibdata1存放在哪个目录下

innodb_log_group_home_dir：　　日志文件ib_logfile0/1存放在哪个目录

innodb_data_file_path：        定义系统表空间文件ibdata1的属性；

innodb_autoextend_increment：  系统表空间文件每次扩展的大小

innodb_log_file_size：          ib_logfile文件大小（写操作多时可以增大）

innodb_log_files_in_group：     有几个ib_logfile文件（写操作多时可以增大 ）

![img]()

**innodb_file_per_table：**  

**关键；开启后，会产生表定义文件 表名.frm，和表数据文件  表名.idb，**

**这样每个表的数据都会存在自己的.idb文件中；如果 关闭，那么所有的**

**数据都会 存在系统表空间文件 ibdata1文件中，这会ibdata1 非常繁忙**

**并且臃肿 庞大，而且ibdata1无法 收缩的，比如线上将一个 大的表 drop掉，**

**此时ibdata1是无法自动缩小的（需要使用 optimiza table 来优化）；而如果开启，数据存在 .idb文件中，则可以随时缩小；**

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201646810-26603526.png)

 

**innodb数据文件存储结构**

 

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201707201-2116954787.png)

 

特点：

- 根据主键寻址速度很快
- 主键值递增的insert插入效率较好
- 主键值随机insert插入操作效率差
- **因此，innodb表必须指定主键，建议使用自增数字；**

如果不使用主键，系统会自动加上一个6字符字符串的主键；

 

innodb数据块缓存池

- 数据的读写需要经过缓存（缓存在buffer pool 即在内存中）
- 数据以整页（16K）位单位读取到缓存中
- 缓存中的数据以LRU策略换出（最少使用策略）
- IO效率高，性能好

![img]()

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201722529-136567453.png)

 

innodb_buffer_pool_size:

![img]()

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201734201-47258135.png)

 

为了IO效率，数据库修改的文件都在内存缓存中完成的；那么我们知道一旦断电，内存中的数据将消失，而数据库是如何保证数据的完整性的呢？那就是数据持久化与事务日志；

 

**innodb 数据持久化与事务日志**

- 事务日志实时持久化
- 内存变化数据（脏数据）增量异步刷出到磁盘
- 实例故障靠重放日志恢复
- 性能好，可靠，恢复快；

![img]()

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201757810-1590929469.png)

 

如果宕机了则：应用已经持久化好了的日志文件，读取日志文件中没有被持久化到数据文件里面的记录；将这些记录重新持久化到我们的数据文件中.

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201813247-838092887.png)

 

![img]()

优缺点：如果实时的刷新到 磁盘中，要找到x随机存放的位置，IO消耗大；而如果将修改刷新到日志文件中，因为它是顺序读写的，速度会快很多。

 

innodb日志持久化相关参数

innodb_flush_log_at_trx_commit 

![img]()

![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201824294-1035907177.png)

 

innodb 行级锁

- 写不阻塞读
- 不同行间的写相互不阻塞
- 并发性能好

 

innodb与事务ACID

事务ACID特性完整支持

- 回滚段失败回滚           A
- 支持主外键约束           C
- 事务版本+回滚段=MVCC    I
- 事务日志持久化           D

 

默认可重复读隔离级别，可以调整

 

**innodb关键特性**

 

- 插入缓冲（insert buffer）
- 两次写（Double write）
- 自适应哈希索引（adaptive hash index）
- 异步io（Async IO）
- 刷新领接页（Flush Neighbor  Page）

带来了更好的性能以及更高的可靠性。

 

 

1.插入缓冲

 

使用场景，即非唯一辅助索引的插入操作，因为不是顺序的，所以将这些插入操作，

存到插入缓冲中去，然后一段时间统一插入到真的索引中去，此时很有可能有几条 插入合并操作，

因为他们是对同一索引页进行的操作，这样就大大提高了效率。

关键就是将一些 离散的操作，缓存起来，然后找到一些对同一个索引页 进行 操作的 操作项 进行合并，这样提高了 效率。

 

 

2.两次写 

提高了 数据页的可靠性。

 

在应用（apply）重做日志前，用户需要一个页的副本，当写入失效发生时，先通过页的副本来还原该页，再进行重做，这就是duble write 。

 ![img](https://images2015.cnblogs.com/blog/820365/201607/820365-20160720201847372-12596408.png)

 

dublewrite组成

- 内存中的dublewrite buffer,大小2M，
- 物理磁盘上共享表空间中连续的128个页，即2个区（extend），大小同样为2M。

对缓冲池的脏页进行刷新时，不是直接写磁盘，而是会通过memcpy()函数将脏页先复制到内存中的doublewrite buffer，

之后通过doublewrite 再分两次，每次1M顺序地写入共享表空间的物理磁盘上，在这个过程中，因为doublewrite页是连续的，

因此这个过程是顺序写的，开销并不是很大。在完成doublewrite页的写入后，再将doublewrite buffer 中的页写入各个 表空间文件中，

此时的写入则是离散的。如果操作系统在将页写入磁盘的过程中发生了崩溃，在恢复过程中，innodb可以从共享表空间中的doublewrite

中找到该页的一个副本，将其复制到表空间文件，再应用重做日志。

 

 

3.自适应哈希索引

哈希（hash）是一种非常快的查找方法，在一般情况下这种查找的时间复杂度为O(1),即一般仅需要一次查找就能定位数据。

而B+树的查找次数，取决于B+树的高度，在生成环境中，B+树的高度一般3-4层，故需要3-4次的查询。

 

innodb会监控对表上个索引页的查询。如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，称之为自适应哈希索引（Adaptive Hash Index，AHI）。

 

AHI有一个要求，就是对这个页的连续访问**模式**必须是一样的。

例如对于（a,b）访问模式情况：

where a = xxx

where a = xxx and b = xxx

 

访问模式一样指的是查询的条件一样，若交替进行上述两种查询，那么innodb不会对该页构造AHI，此外AHI还有如下的要求：

- 以该模式访问了100次
- 页通过该模式访问了N次，其中N=页中记录*1/16;

 

AHI启动后，读写速度提高了2倍，辅助索引的连接操作性能可以提高5倍。

AHI，是数据库自动优化的，DBA只需要指导开发人员去尽量使用符合AHI条件的查询，**以提高效率**。

 

 

4.异步IO

 

sync  IO ：同步IO 即每进行一次IO操作，此次操作结束才能继续接下来的操作。

但是如果用户发需要等待出一条索引扫描的查询，那么这条SQL查询语句可能需要扫描多个索引页，

也就是需要进行多次的IO操作。在每扫描一个页并等待期完成再进行下一次的扫描是没有必要的。

 

异步IO： 

用户可以在发出一个IO请求后立即再发出另一个IO请求，当全部IO请求发送完毕后，等待所有IO操作的完成，这就是AIO。

 

AIO另一个优势可以将多个IO，合并为1个IO，以提高IO效率。例如：

用户需要访问3页内容，但这3页时连续的。同步IO需要进行3次IO,而AIO只需要一次 就可以了。

 

使用AIO的恢复速度 提高了75%

 

5.刷新领接页

工作原理：

当刷新一个脏页时，innodb会检测该页所在区（extent）的所有页，如果是脏页，那么一起进行刷新。

这样做，通过AIO将多个IO写入操作合并为一个IO操作。在传统机械磁盘下有着显著优势。

innodb_flush_neighbors 参数来控制是否开启。

 

 

 **总结**

-  数据库与实例
-  innodb相关磁盘文件：
- -  ibdata1:
  - - 回滚段
    - 表元数据
    - double write
    - insert buffer dump等
  -  ib_logfile0/1
  -  .frm:表定义文件
  -  .ibd:数据文件,innodb_file_per_table=1
-  性能相关参数：
- - innodb_log_file_size
  - innodb_log_files_in_group
  - 原因：当redo log 采用轮寻范式ib_logfile0写完，写ib_logfile1完,清楚ib_logfile0并继续写入ib_logfile0;当ib_logfile1写完，ib_logfile0中还有数据没有持久化到磁盘，又来了新的写入，此时会阻塞新写入，强制刷新ib_logfile0到磁盘，再将新写，写入ib_logfile0;这样就是说，logfile越大其写入越不容易阻塞，写入性能也就越好。
-  数据节点每页16K
-  innodb数据块缓存池
- - 数据读写经过缓存池
  - 数据以整页为单位读取
  - LRU策略（最少使用）换出，
-  innodb数据持久化：通过事务日志
-  innodb_flush_log_at_trx_commit
- - 0：每秒写入并持久化一次（不安全，性能高，无论mysql或服务器宕机，都会丢数据）
  - 1：每次commit都持久化（安全，性能低，IO负担重）
  - 2：每次commit都写入内存的redo log缓存，每秒再刷新到磁盘（安全，性能折中，mysql宕机数据不会丢失，服务器宕机数据会丢失）
-  innodb关键特性
- - 插入缓冲（insert buffer）
  - 两次写（Double write）
  - 自适应哈希索引（adaptive hash index）
  - 异步io（Async IO）
  - 刷新领接页（Flush Neighbor  Page）