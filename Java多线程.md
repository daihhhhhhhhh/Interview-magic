## 线程池

#### ThreadPoolExecutor构造方法



```cpp
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
            ······
                              
    }
```

-  `corePoolSize`: 核心线程数。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，来预创建线程，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中。
-  `maximumPoolSize`：线程池最大线程数。当任务队列满了并且线程数小于`maximumPoolSize`时，线程池会创建新的线程来处理任务。
-  `keepAliveTime`:表示线程没有任务执行时最多保持多久时间会终止。默认情况下，当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，keepAliveTime参数也会作用于核心线程，直到线程池中的线程数为0。
-  `unit`：参数keepAliveTime的时间单位。
-  `workQueue`：任务队列（阻塞队列）。上文已经介绍过了。`ArrayBlockingQueue`和`PriorityBlockingQueue`使用较少，一般使用`LinkedBlockingQueue`和`Synchronous`。线程池的排队策略与BlockingQueue有关。
-  `threadFactory`:线程工厂。可以用线程工厂给每个线程设置名字。通常无需设置该参数。
- `handler`:饱和策略。当任务队列和线程池都满了时采取的应对策略。默认值为`AbortPolicy`，有以下四种取值：

# 拒绝策略

AbortPolicy         -- 当任务添加到线程池中被拒绝时，它将抛出 RejectedExecutionException 异常。
CallerRunsPolicy    -- 当任务添加到线程池中被拒绝时，会在线程池当前正在运行的Thread线程池中处理被拒绝的任务。
DiscardOldestPolicy -- 当任务添加到线程池中被拒绝时，线程池会放弃等待队列中最旧的未处理任务，然后将被拒绝的任务添加到等待队列中。
DiscardPolicy       -- 当任务添加到线程池中被拒绝时，线程池将丢弃被拒绝的任务。

## 阻塞队列

两种常见的阻塞场景：

- 当队列中没有数据的情况下，消费者端的所有线程都会被自动阻塞（挂起），直到有数据放入队列。
- 当队列中填满数据的情况下，生产者端的所有线程都会被自动阻塞（挂起），直到队列中有空的位置，线程被自动唤醒。

### BlockingQueue核心方法

**放入数据**

- `offer(anObject)`:表示如果可能的话,将anObject加到BlockingQueue里,即如果BlockingQueue可以容纳,则返回true,否则返回false.（本方法不阻塞当前执行方法的线程。
- `offer(E o, long timeout, TimeUnit unit)`：可以设定等待的时间，如果在指定的时间内，还不能往队列中加入BlockingQueue，则返回失败。
- `put(anObject)`:把anObject加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断直到BlockingQueue里面有空间再继续。

**获取数据**

- `poll(long timeout, TimeUnit unit)`：从BlockingQueue取出一个队首的对象，如果在指定时间内，队列一旦有数据可取，则立即返回队列中的数据。否则知道时间超时还没有数据可取，返回失败。
- `take()`:取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到BlockingQueue有新的数据被加入;
- `drainTo()`:一次性从BlockingQueue获取所有可用的数据对象（还可以指定获取数据的个数），通过该方法，可以提升获取数据效率；不需要多次分批加锁或释放锁。

### 常见的BlockingQueue

①ArrayBlockingQueue

基于数组的有界阻塞队列，按FIFO排序。新任务进来后，会放到该队列的队尾，有界的数组可以防止资源耗尽问题。当线程池中线程数量达到corePoolSize后，再有新任务进来，则会将任务放入该队列的队尾，等待被调度。如果队列已经是满的，则创建一个新线程，如果线程数量已经达到maxPoolSize，则会执行拒绝策略。

②LinkedBlockingQuene

基于链表的无界阻塞队列（其实最大容量为Interger.MAX），按照FIFO排序。由于该队列的近似无界性，当线程池中线程数量达到corePoolSize后，再有新任务进来，会一直存入该队列，而不会去创建新线程直到maxPoolSize，因此使用该工作队列时，参数maxPoolSize其实是不起作用的。

③SynchronousQuene

一个不缓存任务的阻塞队列，生产者放入一个任务必须等到消费者取出这个任务。也就是说新任务进来时，不会缓存，而是直接被调度执行该任务，如果没有可用线程，则创建新线程，如果线程数量达到maxPoolSize，则执行拒绝策略。

其实就是相当于没有使用队列缓存,在达到最大线程数之前,每一个任务进来都会直接交给空闲线程被执行.没有空闲线程则创建新线程,如果线程数超出最大线程数,则执行拒绝策略.

④PriorityBlockingQueue

具有优先级的无界阻塞队列，优先级通过参数Comparator实现。前两个的队列都是按照先进先出FIFO顺序执行的,而优先级队列是根据优先级的值大小来执行的

⑤DelayQueue

支持延时获取元素的无界阻塞队列。队列中每个元素必须实现Delayed接口。插入数据的操作（生产者）永远不会被阻塞，只有获取数据的操作（消费者）才会被阻塞。

### 如何设置参数

  1、默认值
    \* corePoolSize=1
    \* queueCapacity=Integer.MAX_VALUE
    \* maxPoolSize=Integer.MAX_VALUE
    \* keepAliveTime=60s
    \* allowCoreThreadTimeout=false
    \* rejectedExecutionHandler=AbortPolicy()

  2、如何来设置
    \* 需要根据几个值来决定
      \- tasks ：每秒的任务数，假设为500~1000
      \- taskcost：每个任务花费时间，假设为0.1s
      \- responsetime：系统允许容忍的最大响应时间，假设为1s
    \* 做几个计算
      \- corePoolSize = 每秒需要多少个线程处理？ 
        \* threadcount = tasks/(1/taskcost) =tasks*taskcout = (500~1000)*0.1 = 50~100 个线程。corePoolSize设置应该大于50
        \* 根据8020原则，如果80%的每秒任务数小于800，那么corePoolSize设置为80即可
      \- queueCapacity = (coreSizePool/taskcost)*responsetime
        \* 计算可得 queueCapacity = 80/0.1*1 = 800。意思是队列里的线程可以等待1s，超过了的需要新开线程来执行
        \* 切记不能设置为Integer.MAX_VALUE，这样队列会很大，线程数只会保持在corePoolSize大小，当任务陡增时，不能新开线程来执行，响应时间会随之陡增。
      \- maxPoolSize = (max(tasks)- queueCapacity)/(1/taskcost)
        \* 计算可得 maxPoolSize = (1000-800)/10 = 20
        \* （最大任务数-队列容量）/每个线程每秒处理能力 = 最大线程数
      \- rejectedExecutionHandler：根据具体情况来决定，任务不重要可丢弃，任务重要则要利用一些缓冲机制来处理
      \- keepAliveTime和allowCoreThreadTimeout采用默认通常能满足

  3、 以上都是理想值，实际情况下要根据机器性能来决定。如果在未达到最大线程数的情况机器cpu load已经满了，则需要通过升级硬件（呵呵）和优化代码，降低taskcost来处理。

