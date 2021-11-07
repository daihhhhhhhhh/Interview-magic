# Java并发之AQS详解

# 一、概述

　　谈到并发，不得不谈ReentrantLock；而谈到ReentrantLock，不得不谈AbstractQueuedSynchronizer（AQS）！

　　类如其名，抽象的队列式的同步器，AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，如常用的ReentrantLock/Semaphore/CountDownLatch...。

　　以下是本文的目录大纲：

1. 1. 概述
   2. 框架
   3. 源码详解
   4. 简单应用

　　若有不正之处，请谅解和批评指正，不胜感激。

　　请尊重作者劳动成果，转载请标明原文链接（原文持续更新，建议阅读原文）：http://www.cnblogs.com/waterystone/p/4920797.html

# 二、框架

![img](https://images2015.cnblogs.com/blog/721070/201705/721070-20170504110246211-10684485.png)

　　它维护了一个volatile int state（代表共享资源）和一个FIFO线程等待队列（多线程争用资源被阻塞时会进入此队列）。这里volatile是核心关键词，具体volatile的语义，在此不述。state的访问方式有三种:

- getState()
- setState()
- compareAndSetState()

　　AQS定义两种资源共享方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）。

　　不同的自定义同步器争用共享资源的方式也不同。**自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可**，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

　　以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

　　再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

　　一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。

# 三、源码详解

　　本节开始讲解AQS的源码实现。依照acquire-release、acquireShared-releaseShared的次序来。

## 3.0 结点状态waitStatus

   这里我们说下Node。Node结点是对每一个等待获取资源的线程的封装，其包含了需要同步的线程本身及其等待状态，如是否被阻塞、是否等待唤醒、是否已经被取消等。变量waitStatus则表示当前Node结点的等待状态，共有5种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE、0。

- **CANCELLED**(1)：表示当前结点已取消调度。当timeout或被中断（响应中断的情况下），会触发变更为此状态，进入该状态后的结点将不会再变化。
- **SIGNAL**(-1)：表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL。
- **CONDITION**(-2)：表示结点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将**从等待队列转移到同步队列中**，等待获取同步锁。
- **PROPAGATE**(-3)：共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
- **0**：新结点入队时的默认状态。

注意，**负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用>0、<0来判断结点的状态是否正常**。

## 3.1 acquire(int)

　　此方法是独占模式下线程获取共享资源的顶层入口。如果获取到资源，线程直接返回，否则进入等待队列，直到获取到资源为止，且整个过程忽略中断的影响。这也正是lock()的语义，当然不仅仅只限于lock()。获取到资源后，线程就可以去执行其临界区代码了。下面是acquire()的源码：

```
1 public final void acquire(int arg) {
2     if (!tryAcquire(arg) &&
3         acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
4         selfInterrupt();
5 }
```

 

　　函数流程如下：

1. 1. tryAcquire()尝试直接去获取资源，如果成功则直接返回（这里体现了非公平锁，每个线程获取锁时会尝试直接抢占加塞一次，而CLH队列中可能还有别的线程在等待）；
   2. addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
   3. acquireQueued()使线程阻塞在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
   4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

　　这时单凭这4个抽象的函数来看流程还有点朦胧，不要紧，看完接下来的分析后，你就会明白了。就像《大话西游》里唐僧说的：等你明白了舍生取义的道理，你自然会回来和我唱这首歌的。

### 3.1.1 tryAcquire(int)

　　此方法尝试去获取独占资源。如果获取成功，则直接返回true，否则直接返回false。这也正是tryLock()的语义，还是那句话，当然不仅仅只限于tryLock()。如下是tryAcquire()的源码：

```
1     protected boolean tryAcquire(int arg) {
2         throw new UnsupportedOperationException();
3     }
```

 

　　什么？直接throw异常？说好的功能呢？好吧，**还记得概述里讲的AQS只是一个框架，具体资源的获取/释放方式交由自定义同步器去实现吗？**就是这里了！！！AQS这里只定义了一个接口，具体资源的获取交由自定义同步器去实现了（通过state的get/set/CAS）！！！至于能不能重入，能不能加塞，那就看具体的自定义同步器怎么去设计了！！！当然，自定义同步器在进行资源访问时要考虑线程安全的影响。

　　这里之所以没有定义成abstract，是因为独占模式下只用实现tryAcquire-tryRelease，而共享模式下只用实现tryAcquireShared-tryReleaseShared。如果都定义成abstract，那么每个模式也要去实现另一模式下的接口。说到底，Doug Lea还是站在咱们开发者的角度，尽量减少不必要的工作量。

### 3.1.2 addWaiter(Node)

　　此方法用于将当前线程加入到等待队列的队尾，并返回当前线程所在的结点。还是上源码吧：


```
 1 private Node addWaiter(Node mode) {
 2     //以给定模式构造结点。mode有两种：EXCLUSIVE（独占）和SHARED（共享）
 3     Node node = new Node(Thread.currentThread(), mode);
 4     
 5     //尝试快速方式直接放到队尾。
 6     Node pred = tail;
 7     if (pred != null) {
 8         node.prev = pred;
 9         if (compareAndSetTail(pred, node)) {
10             pred.next = node;
11             return node;
12         }
13     }
14     
15     //上一步失败则通过enq入队。
16     enq(node);
17     return node;
18 }
```



 不用再说了，直接看注释吧。

#### 3.1.2.1 enq(Node)

 　此方法用于将node加入队尾。源码如下：



```
 1 private Node enq(final Node node) {
 2     //CAS"自旋"，直到成功加入队尾
 3     for (;;) {
 4         Node t = tail;
 5         if (t == null) { // 队列为空，创建一个空的标志结点作为head结点，并将tail也指向它。
 6             if (compareAndSetHead(new Node()))
 7                 tail = head;
 8         } else {//正常流程，放入队尾
 9             node.prev = t;
10             if (compareAndSetTail(t, node)) {
11                 t.next = node;
12                 return t;
13             }
14         }
15     }
16 }
```



 

如果你看过AtomicInteger.getAndIncrement()函数源码，那么相信你一眼便看出这段代码的精华。**CAS自旋volatile变量**，是一种很经典的用法。还不太了解的，自己去百度一下吧。

### 3.1.3 acquireQueued(Node, int)

　　OK，通过tryAcquire()和addWaiter()，该线程获取资源失败，已经被放入等待队列尾部了。聪明的你立刻应该能想到该线程下一部该干什么了吧：**进入等待状态休息，直到其他线程彻底释放资源后唤醒自己，自己再拿到资源，然后就可以去干自己想干的事了**。没错，就是这样！是不是跟医院排队拿号有点相似~~acquireQueued()就是干这件事：**在等待队列中排队拿号（中间没其它事干可以休息），直到拿到号后再返回**。这个函数非常关键，还是上源码吧：



```
 1 final boolean acquireQueued(final Node node, int arg) {
 2     boolean failed = true;//标记是否成功拿到资源
 3     try {
 4         boolean interrupted = false;//标记等待过程中是否被中断过
 5         
 6         //又是一个“自旋”！
 7         for (;;) {
 8             final Node p = node.predecessor();//拿到前驱
 9             //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
10             if (p == head && tryAcquire(arg)) {
11                 setHead(node);//拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
12                 p.next = null; // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
13                 failed = false; // 成功获取资源
14                 return interrupted;//返回等待过程中是否被中断过
15             }
16             
17             //如果自己可以休息了，就通过park()进入waiting状态，直到被unpark()。如果不可中断的情况下被中断了，那么会从park()中醒过来，发现拿不到资源，从而继续进入park()等待。
18             if (shouldParkAfterFailedAcquire(p, node) &&
19                 parkAndCheckInterrupt())
20                 interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
21         }
22     } finally {
23         if (failed) // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待。
24             cancelAcquire(node);
25     }
26 }
```


　 

到这里了，我们先不急着总结acquireQueued()的函数流程，先看看shouldParkAfterFailedAcquire()和parkAndCheckInterrupt()具体干些什么。

#### 3.1.3.1 shouldParkAfterFailedAcquire(Node, Node)

　　此方法主要用于检查状态，看看自己是否真的可以去休息了（进入waiting状态，如果线程状态转换不熟，可以参考本人上一篇写的[Thread详解](http://www.cnblogs.com/waterystone/p/4920007.html)），万一队列前边的线程都放弃了只是瞎站着，那也说不定，对吧！



```
 1 private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
 2     int ws = pred.waitStatus;//拿到前驱的状态
 3     if (ws == Node.SIGNAL)
 4         //如果已经告诉前驱拿完号后通知自己一下，那就可以安心休息了
 5         return true;
 6     if (ws > 0) {
 7         /*
 8          * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
 9          * 注意：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被保安大叔赶走了(GC回收)！
10          */
11         do {
12             node.prev = pred = pred.prev;
13         } while (pred.waitStatus > 0);
14         pred.next = node;
15     } else {
16          //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
17         compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
18     }
19     return false;
20 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

整个流程中，如果前驱结点的状态不是SIGNAL，那么自己就不能安心去休息，需要去找个安心的休息点，同时可以再尝试下看有没有机会轮到自己拿号。

#### 3.1.3.2 parkAndCheckInterrupt()

　　如果线程找好安全休息点后，那就可以安心去休息了。此方法就是让线程去休息，真正进入等待状态。

```
1 private final boolean parkAndCheckInterrupt() {
2     LockSupport.park(this);//调用park()使线程进入waiting状态
3     return Thread.interrupted();//如果被唤醒，查看自己是不是被中断的。
4 }
```

 　park()会让当前线程进入waiting状态。在此状态下，有两种途径可以唤醒该线程：1）被unpark()；2）被interrupt()。（再说一句，如果线程状态转换不熟，可以参考本人写的[Thread详解](http://www.cnblogs.com/waterystone/p/4920007.html)）。需要注意的是，Thread.interrupted()会清除当前线程的中断标记位。 

#### 3.1.3.3 小结

　　OK，看了shouldParkAfterFailedAcquire()和parkAndCheckInterrupt()，现在让我们再回到acquireQueued()，总结下该函数的具体流程：

1. 结点进入队尾后，检查状态，找到安全休息点；
2. 调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
3. 被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。

 

### 3.1.4 小结

　　OKOK，acquireQueued()分析完之后，我们接下来再回到acquire()！再贴上它的源码吧：

```
1 public final void acquire(int arg) {
2     if (!tryAcquire(arg) &&
3         acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
4         selfInterrupt();
5 }
```

再来总结下它的流程吧：

1. 调用自定义同步器的tryAcquire()尝试直接去获取资源，如果成功则直接返回；
2. 没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

由于此函数是重中之重，我再用流程图总结一下：

![img](https://images2015.cnblogs.com/blog/721070/201511/721070-20151102145743461-623794326.png)

至此，acquire()的流程终于算是告一段落了。这也就是ReentrantLock.lock()的流程，不信你去看其lock()源码吧，整个函数就是一条acquire(1)！！！

 

## 3.2 release(int)

 　上一小节已经把acquire()说完了，这一小节就来讲讲它的反操作release()吧。此方法是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。这也正是unlock()的语义，当然不仅仅只限于unlock()。下面是release()的源码：



```
1 public final boolean release(int arg) {
2     if (tryRelease(arg)) {
3         Node h = head;//找到头结点
4         if (h != null && h.waitStatus != 0)
5             unparkSuccessor(h);//唤醒等待队列里的下一个线程
6         return true;
7     }
8     return false;
9 }
```



 

　　逻辑并不复杂。它调用tryRelease()来释放资源。有一点需要注意的是，**它是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自定义同步器在设计tryRelease()的时候要明确这一点！！**

### 3.2.1 tryRelease(int)

　　此方法尝试去释放指定量的资源。下面是tryRelease()的源码：

```
1 protected boolean tryRelease(int arg) {
2     throw new UnsupportedOperationException();
3 }
```

 

　　跟tryAcquire()一样，这个方法是需要独占模式的自定义同步器去实现的。正常来说，tryRelease()都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(state-=arg)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，**release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！**所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false。

### 3.2.2 unparkSuccessor(Node)

　　此方法用于唤醒等待队列中下一个线程。下面是源码：



```
 1 private void unparkSuccessor(Node node) {
 2     //这里，node一般为当前线程所在的结点。
 3     int ws = node.waitStatus;
 4     if (ws < 0)//置零当前线程所在的结点状态，允许失败。
 5         compareAndSetWaitStatus(node, ws, 0);
 6 
 7     Node s = node.next;//找到下一个需要唤醒的结点s
 8     if (s == null || s.waitStatus > 0) {//如果为空或已取消
 9         s = null;
10         for (Node t = tail; t != null && t != node; t = t.prev) // 从后向前找。
11             if (t.waitStatus <= 0)//从这里可以看出，<=0的结点，都是还有效的结点。
12                 s = t;
13     }
14     if (s != null)
15         LockSupport.unpark(s.thread);//唤醒
16 }
```



 

　　这个函数并不复杂。一句话概括：**用unpark()唤醒等待队列中最前边的那个未放弃线程**，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了！！And then, DO what you WANT!

### 3.2.3 小结

　　release()是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。

 

   74楼的朋友提了一个非常有趣的问题：如果获取锁的线程在release时异常了，没有unpark队列中的其他结点，这时队列中的其他结点会怎么办？是不是没法再被唤醒了？

   答案是**YES**（测试程序详见76楼）！！！这时，队列中等待锁的线程将永远处于park状态，无法再被唤醒！！！但是我们再回头想想，获取锁的线程在什么情形下会release抛出异常呢？？

1. 线程突然死掉了？可以通过thread.stop来停止线程的执行，但该函数的执行条件要严苛的多，而且函数注明是非线程安全的，已经标明Deprecated；
2. 线程被interupt了？线程在运行态是不响应中断的，所以也不会抛出异常；
3. release代码有bug，抛出异常了？目前来看，Doug Lea的release方法还是比较健壮的，没有看出能引发异常的情形（如果有，恐怕早被用户吐槽了）。**除非自己写的tryRelease()有bug，那就没啥说的，自己写的bug只能自己含着泪去承受了**。

## 3.3 acquireShared(int)

　　此方法是共享模式下线程获取共享资源的顶层入口。它会获取指定量的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止，整个过程忽略中断。下面是acquireShared()的源码：

```
1 public final void acquireShared(int arg) {
2     if (tryAcquireShared(arg) < 0)
3         doAcquireShared(arg);
4 }
```

 

　　这里tryAcquireShared()依然需要自定义同步器去实现。但是AQS已经把其返回值的语义定义好了：负值代表获取失败；0代表获取成功，但没有剩余资源；正数表示获取成功，还有剩余资源，其他线程还可以去获取。所以这里acquireShared()的流程就是：

1. 1. tryAcquireShared()尝试获取资源，成功则直接返回；
   2. 失败则通过doAcquireShared()进入等待队列，直到获取到资源为止才返回。

### 3.3.1 doAcquireShared(int)

　　此方法用于将当前线程加入等待队列尾部休息，直到其他线程释放资源唤醒自己，自己成功拿到相应量的资源后才返回。下面是doAcquireShared()的源码：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 private void doAcquireShared(int arg) {
 2     final Node node = addWaiter(Node.SHARED);//加入队列尾部
 3     boolean failed = true;//是否成功标志
 4     try {
 5         boolean interrupted = false;//等待过程中是否被中断过的标志
 6         for (;;) {
 7             final Node p = node.predecessor();//前驱
 8             if (p == head) {//如果到head的下一个，因为head是拿到资源的线程，此时node被唤醒，很可能是head用完资源来唤醒自己的
 9                 int r = tryAcquireShared(arg);//尝试获取资源
10                 if (r >= 0) {//成功
11                     setHeadAndPropagate(node, r);//将head指向自己，还有剩余资源可以再唤醒之后的线程
12                     p.next = null; // help GC
13                     if (interrupted)//如果等待过程中被打断过，此时将中断补上。
14                         selfInterrupt();
15                     failed = false;
16                     return;
17                 }
18             }
19             
20             //判断状态，寻找安全点，进入waiting状态，等着被unpark()或interrupt()
21             if (shouldParkAfterFailedAcquire(p, node) &&
22                 parkAndCheckInterrupt())
23                 interrupted = true;
24         }
25     } finally {
26         if (failed)
27             cancelAcquire(node);
28     }
29 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

　　有木有觉得跟acquireQueued()很相似？对，其实流程并没有太大区别。只不过这里将补中断的selfInterrupt()放到doAcquireShared()里了，而独占模式是放到acquireQueued()之外，其实都一样，不知道Doug Lea是怎么想的。

　　跟独占模式比，还有一点需要注意的是，这里只有线程是head.next时（“老二”），才会去尝试获取资源，有剩余的话还会唤醒之后的队友。那么问题就来了，假如老大用完后释放了5个资源，而老二需要6个，老三需要1个，老四需要2个。老大先唤醒老二，老二一看资源不够，他是把资源让给老三呢，还是不让？答案是否定的！老二会继续park()等待其他线程释放资源，也更不会去唤醒老三和老四了。独占模式，同一时刻只有一个线程去执行，这样做未尝不可；但共享模式下，多个线程是可以同时执行的，现在因为老二的资源需求量大，而把后面量小的老三和老四也都卡住了。当然，这并不是问题，只是AQS保证严格按照入队顺序唤醒罢了（保证公平，但降低了并发）。

 

#### 3.3.1.1 setHeadAndPropagate(Node, int)



```
 1 private void setHeadAndPropagate(Node node, int propagate) {
 2     Node h = head; 
 3     setHead(node);//head指向自己
 4      //如果还有剩余量，继续唤醒下一个邻居线程
 5     if (propagate > 0 || h == null || h.waitStatus < 0) {
 6         Node s = node.next;
 7         if (s == null || s.isShared())
 8             doReleaseShared();
 9     }
10 }
```



 

　　此方法在setHead()的基础上多了一步，就是自己苏醒的同时，如果条件符合（比如还有剩余资源），还会去唤醒后继结点，毕竟是共享模式！

　　doReleaseShared()我们留着下一小节的releaseShared()里来讲。

 

### 3.3.2 小结

　　OK，至此，acquireShared()也要告一段落了。让我们再梳理一下它的流程：

1. 

2. 1. tryAcquireShared()尝试获取资源，成功则直接返回；
   2. 失败则通过doAcquireShared()进入等待队列park()，直到被unpark()/interrupt()并成功获取到资源才返回。整个等待过程也是忽略中断的。

　　其实跟acquire()的流程大同小异，只不过多了个**自己拿到资源后，还会去唤醒后继队友的操作（这才是共享嘛）**。

## 3.4 releaseShared()

　　上一小节已经把acquireShared()说完了，这一小节就来讲讲它的反操作releaseShared()吧。此方法是共享模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其他线程来获取资源。下面是releaseShared()的源码：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
1 public final boolean releaseShared(int arg) {
2     if (tryReleaseShared(arg)) {//尝试释放资源
3         doReleaseShared();//唤醒后继结点
4         return true;
5     }
6     return false;
7 }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

　　此方法的流程也比较简单，一句话：释放掉资源后，唤醒后继。跟独占模式下的release()相似，但有一点稍微需要注意：独占模式下的tryRelease()在完全释放掉资源（state=0）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。例如，资源总量是13，A（5）和B（7）分别获取到资源并发运行，C（4）来时只剩1个资源就需要等待。A在运行过程中释放掉2个资源量，然后tryReleaseShared(2)返回true唤醒C，C一看只有3个仍不够继续等待；随后B又释放2个，tryReleaseShared(2)返回true唤醒C，C一看有5个够自己用了，然后C就可以跟A和B一起运行。而ReentrantReadWriteLock读锁的tryReleaseShared()只有在完全释放掉资源（state=0）才返回true，所以自定义同步器可以根据需要决定tryReleaseShared()的返回值。

### 3.4.1 doReleaseShared()

　　此方法主要用于唤醒后继。下面是它的源码：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 1 private void doReleaseShared() {
 2     for (;;) {
 3         Node h = head;
 4         if (h != null && h != tail) {
 5             int ws = h.waitStatus;
 6             if (ws == Node.SIGNAL) {
 7                 if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
 8                     continue;
 9                 unparkSuccessor(h);//唤醒后继
10             }
11             else if (ws == 0 &&
12                      !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
13                 continue;
14         }
15         if (h == head)// head发生变化
16             break;
17     }
18 }
```



 

 

## 3.5 小结

　　本节我们详解了独占和共享两种模式下获取-释放资源(acquire-release、acquireShared-releaseShared)的源码，相信大家都有一定认识了。值得注意的是，acquire()和acquireShared()两种方法下，线程在等待队列中都是忽略中断的。AQS也支持响应中断的，acquireInterruptibly()/acquireSharedInterruptibly()即是，相应的源码跟acquire()和acquireShared()差不多，这里就不再详解了。

 

# 四、简单应用

　　通过前边几个章节的学习，相信大家已经基本理解AQS的原理了。这里再将“框架”一节中的一段话复制过来：

　　不同的自定义同步器争用共享资源的方式也不同。**自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可**，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

　　OK，下面我们就以AQS源码里的Mutex为例，讲一下AQS的简单应用。

## 4.1 Mutex（互斥锁）

　　Mutex是一个不可重入的互斥锁实现。锁资源（AQS里的state）只有两种状态：0表示未锁定，1表示锁定。下边是Mutex的核心源码：



```
 1 class Mutex implements Lock, java.io.Serializable {
 2     // 自定义同步器
 3     private static class Sync extends AbstractQueuedSynchronizer {
 4         // 判断是否锁定状态
 5         protected boolean isHeldExclusively() {
 6             return getState() == 1;
 7         }
 8 
 9         // 尝试获取资源，立即返回。成功则返回true，否则false。
10         public boolean tryAcquire(int acquires) {
11             assert acquires == 1; // 这里限定只能为1个量
12             if (compareAndSetState(0, 1)) {//state为0才设置为1，不可重入！
13                 setExclusiveOwnerThread(Thread.currentThread());//设置为当前线程独占资源
14                 return true;
15             }
16             return false;
17         }
18 
19         // 尝试释放资源，立即返回。成功则为true，否则false。
20         protected boolean tryRelease(int releases) {
21             assert releases == 1; // 限定为1个量
22             if (getState() == 0)//既然来释放，那肯定就是已占有状态了。只是为了保险，多层判断！
23                 throw new IllegalMonitorStateException();
24             setExclusiveOwnerThread(null);
25             setState(0);//释放资源，放弃占有状态
26             return true;
27         }
28     }
29 
30     // 真正同步类的实现都依赖继承于AQS的自定义同步器！
31     private final Sync sync = new Sync();
32 
33     //lock<-->acquire。两者语义一样：获取资源，即便等待，直到成功才返回。
34     public void lock() {
35         sync.acquire(1);
36     }
37 
38     //tryLock<-->tryAcquire。两者语义一样：尝试获取资源，要求立即返回。成功则为true，失败则为false。
39     public boolean tryLock() {
40         return sync.tryAcquire(1);
41     }
42 
43     //unlock<-->release。两者语文一样：释放资源。
44     public void unlock() {
45         sync.release(1);
46     }
47 
48     //锁是否占有状态
49     public boolean isLocked() {
50         return sync.isHeldExclusively();
51     }
52 }
```



 

　　同步类在实现时一般都将自定义同步器（sync）定义为内部类，供自己使用；而同步类自己（Mutex）则实现某个接口，对外服务。当然，接口的实现要直接依赖sync，它们在语义上也存在某种对应关系！！而sync只用实现资源state的获取-释放方式tryAcquire-tryRelelase，至于线程的排队、等待、唤醒等，上层的AQS都已经实现好了，我们不用关心。

　　除了Mutex，ReentrantLock/CountDownLatch/Semphore这些同步类的实现方式都差不多，不同的地方就在获取-释放资源的方式tryAcquire-tryRelelase。掌握了这点，AQS的核心便被攻破了！