# ConcurrentHashMap扩容机制

ConcurrentHashMap，jdk1.8，采用多线程扩容。整个扩容过程，通过CAS设置sizeCtl、transferIndex等变量协调多个线程进行并发扩容。多线程无锁扩容的关键就是通过CAS设置sizeCtl与transferIndex变量，协调多个线程对table数组中的node进行迁移。

### 如何实现线程安全

采用CAS和synchronized关键字来保证线程安全。

### put方法逻辑

1. 计算key的hash值
2. 判断Node[]数组是否初始化，没有则进行初始化操作
3. 通过hash定位数组的索引坐标，是否有Node节点，如果没有则使用CAS进行添加（链表的头节点），添加失败则进入下次循环。
4. 检查到内部正在扩容，就帮助它一块扩容。
5. 如果f！=null，则使用synchronized锁住f元素（链表/红黑树的头元素）。如果是Node（链表结构）则执行链表的添加操作；如果是TreeNode（树型结构）则执行树添加操作。
6. 判断链表长度已经达到临界值8（默认值），当节点超过这个值就需要把链表转换为树结构。
7. 如果添加成功就调用addCount（）方法统计size，并且检查是否需要扩容

### 扩容过程分析

1、线程执行put操作，发现容量已经达到扩容阈值，需要进行扩容操作，此时transferindex=tab.length=32
2、扩容线程A 以CAS的方式修改transferindex=31-16=16 ,然后按照降序迁移table[31]至table[16]这个区间的hash桶
3、迁移hash桶时，会将桶内的链表或者红黑树，按照一定算法，拆分成2份，将其插入nextTable[i]和nextTable[i+n]（n是table数组的长度）。 迁移完毕的hash桶,会被设置成ForwardingNode节点，以此告知访问此桶的其他线程，此节点已经迁移完毕。
4、此时线程2访问到了ForwardingNode节点，如果线程2执行的put或remove等写操作，那么就会先帮其扩容。如果线程2执行的是get等读方法，则会调用ForwardingNode的find方法，去nextTable里面查找相关元素。

#### sizeCtl属性

```
private transient volatile int sizeCtl;
1
```

多线程之间，以volatile的方式读取sizeCtl属性，来判断ConcurrentHashMap当前所处的状态。通过CAS设置sizeCtl属性，告知其他线程ConcurrentHashMap的状态变更。

```java
不同状态，sizeCtl所代表的含义也有所不同。
未初始化：sizeCtl=0：表示没有指定初始容量。sizeCtl>0：表示初始容量。
初始化中：sizeCtl=-1,标记作用，告知其他线程，正在初始化
正常状态：sizeCtl=0.75n,扩容阈值
扩容中：sizeCtl < 0 : 表示有其他线程正在执行扩容
sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT)+2 表示此时只有一个线程在执行扩容
```

#### transferIndex 扩容索引

扩容索引，表示已经分配给扩容线程的table数组索引位置。主要用来协调多个线程，并发安全地获取迁移任务（hash桶）。

```
private transient volatile int transferIndex;
private static final int MIN_TRANSFER_STRIDE = 16; //扩容线程每次最少要迁移16个hash桶
12
```

1、在扩容之前，transferIndex 在数组的最右边 。此时有一个线程发现已经到达扩容阈值，准备开始扩容。
2、扩容线程，在迁移数据之前，首先要将transferIndex右移（以CAS的方式修改 transferIndex=transferIndex-stride(要迁移hash桶的个数)），获取迁移任务。每个扩容线程都会通过for循环+CAS的方式设置transferIndex，因此可以确保多线程扩容的并发安全。

换个角度，我们可以将待迁移的table数组，看成一个任务队列，transferIndex看成任务队列的头指针。而扩容线程，就是这个队列的消费者。扩容线程通过CAS设置transferIndex索引的过程，就是消费者从任务队列中获取任务的过程。为了性能考虑，我们当然不会每次只获取一个任务（hash桶），因此ConcurrentHashMap规定，每次至少要获取16个迁移任务（迁移16个hash桶，MIN_TRANSFER_STRIDE = 16）

### 无锁的执行者-CAS

CAS的全称是Compare And Swap 即比较交换，其算法核心思想如下

```java
执行函数：CAS(V,E,N)
其包含3个参数：V表示要更新的变量；E表示预期值；N表示新值
```

如果V值等于E值，则将V的值设为N。若V值和E值不同，则说明已经有其他线程做了更新，则当前线程什么都不做。
通俗的理解就是CAS操作需要我们提供一个期望值，当期望值与当前线程的变量值相同时，说明还没线程修改该值，当前线程可以进行修改，也就是执行CAS操作，但如果期望值与当前线程不符，则说明该值已被其他线程修改，此时不执行更新操作，但可以选择重新读取该变量再尝试再次修改该变量，也可以放弃操作。

由于CAS操作属于乐观派，它总认为自己可以成功完成操作，当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作，这点从图中也可以看出来。基于这样的原理，CAS操作即使没有锁，同样知道其他线程对共享资源操作影响，并执行相应的处理措施。

同时从这点也可以看出，由于无锁操作中没有锁的存在，因此不可能出现死锁的情况，也就是说无锁操作天生免疫死锁。

### 为什么要使用CAS+Synchronized取代Segment+ReentrantLock

假设你对CAS,Synchronized,ReentrantLock这些知识很了解,并且知道AQS,自旋锁,偏向锁,轻量级锁,重量级锁这些知识,也知道Synchronized和ReentrantLock在唤醒被挂起线程竞争的时候有什么区别。
Synchronized上锁的对象,请记住,Synchronized是靠对象的对象头和此对象对应的monitor来保证上锁的,也就是对象头里的重量级锁标志指向了monitor,而monitor内部则保存了一个当前线程,也就是抢到了锁的线程.

那么这里的 f 是什么呢?**它是Node链表里的每一个node,也就是说,Synchronized是将每一个node对象作为了一个锁,这样做的好处是将锁细化了**,也就是说,除非两个线程同时操作一个node,注意,是一个node而不是一个Node链表,那么才会争抢同一把锁.

如果使用ReentrantLock其实也可以将锁细化成这样的,只要让Node类继承ReentrantLock就行了,这样的话调用f.lock()就能做到和Synchronized(f)同样的效果,但为什么不这样做呢?

试想一下,锁已经被细化到这种程度了,那么出现并发争抢的可能性还高吗?哪怕出现争抢了,只要线程可以在30到50次自旋里拿到锁,**那么Synchronized就不会升级为重量级锁,而等待的线程也就不用被挂起,我们也就少了挂起和唤醒这个上下文切换的过程开销**.

但如果是ReentrantLock,它只有在线程没有抢到锁,然后新建Node节点后再尝试一次而已,**不会自旋,而是直接被挂起,这样一来就很容易多出线程上下文开销的代价**.当然,你也可以使用tryLock(),但是这样又出现了一个问题,你怎么知道tryLock的时间呢?在时间范围里还好,假如超过了呢?

所以,在锁被细化到如此程度上,使用Synchronized是最好的选择了.这里再补充一句,Synchronized和ReentrantLock他们的开销差距是在释放锁时唤醒线程的数量,Synchronized是唤醒锁池里所有的线程+刚好来访问的线程,而ReentrantLock则是当前线程后进来的第一个线程+刚好来访问的线程.

如果是线程并发量不大的情况下,那么Synchronized因为自旋锁,偏向锁,轻量级锁的原因,不用将等待线程挂起,偏向锁甚至不用自旋,所以在这种情况下要比ReentrantLock高效.