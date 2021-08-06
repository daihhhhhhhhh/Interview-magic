# ReentrantLock的底层原理

第一篇专栏总结一下Java自带的可重入锁

```java
public class ReentrantLock implements Lock, Serializable
```

ReentrantLock底层使用了CAS+AQS队列实现，下面分别具体介绍两个技术。

## **1. CAS**（Compare and Swap）

CAS是一种无锁算法。有3个操作数：内存值V、旧的预期值A、要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

```text
do{
  备份旧数据；
  基于旧数据构造新数据；
}while(!CAS( 内存地址，备份的旧数据，新数据 ))
```

该操作是一个原子操作，被广泛的应用在Java的底层实现中。

在Java中，CAS主要是由sun.misc.Unsafe这个类通过JNI调用CPU底层指令实现。

### 1.1 CAS的开销

CAS速度非常快：

1. CAS是CPU指令级的操作，只有一步原子操作；
2. CAS避免了请求操作系统来裁定锁的问题，不用麻烦操作系统，直接在CPU内部就搞定了

CAS仍然可能有消耗：可能出现cache miss的情况，会有更大的cpu时间消耗。

![img](https://pic2.zhimg.com/80/v2-c5bd0cb47a635a41cabd45e85c97259d_1440w.jpg)

### 2. AQS队列

AQS是一个用于构建锁和同步容器的框架。

AQS使用一个FIFO的队列（也叫CLH队列，是[CLH锁](https://link.zhihu.com/?target=https%3A//blog.csdn.net/claram/article/details/83828768)的一种变形），表示排队**等待锁的线程**。队列**头节点**称作“哨兵节点”或者“哑节点”，它不与任何线程关联。**其他的节点**与等待线程关联，每个节点维护一个等待状态waitStatus。结构如下图所示：

![img](https://pic1.zhimg.com/80/v2-dcac46a22d27e93ab38511c4c2a63504_1440w.jpg)

### 3. ReentrantLock的流程

1. ReentrantLock先通过CAS尝试获取锁，

2. 1. 如果此时锁已经被占用，该线程加入AQS队列并wait()

   2. 当前驱线程的锁被释放，挂在CLH队列为首的线程就会被notify()，然后继续CAS尝试获取锁，此时：

   3. 1. 非公平锁，如果有其他线程尝试lock()，有可能被其他刚好申请锁的线程**抢占**。
      2. 公平锁，只有在CLH**队列头的线程**才可以获取锁，**新来的线程**只能插入到队尾。

（注：ReentrantLock默认是非公平锁，也可以指定为公平锁）

ReentrantLock的2个构造函数

```java
public ReentrantLock() {
    sync = new NonfairSync(); //默认，非公平
}
 
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync(); //根据参数创建
}
```

### 4. lock() 和 unlock() 的实现

### 4.1 lock()函数

如果成功通过CAS修改了state，指定当前线程为该锁的独占线程，标志自己成功获取锁。

如果CAS失败的话，调用acquire();

```java
    final void lock() { //非公平锁
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    final void lock() { //公平锁
            acquire(1);
    }
```

acquire()函数：

首先，调用tryAcquire()，会尝试再次通过CAS修改state为1，

如果失败而且发现锁是被当前线程占用的，就执行重入（state++）；

如果锁是被其他线程占有，那么当前线程执行tryAcquire返回失败，并且执行addWaiter()进入等待队列，并挂起自己interrupt()。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

4.2 tryAcquire()

检查state字段，

若为0，表示锁未被占用，那么尝试占用；

若不为0，检查当前锁是否被自己占用，若被自己占用，则更新state字段，表示**重入锁的次数**。

如果以上两点都没有成功，则获取锁失败，返回false。

```java
 protected final boolean tryAcquire(int acquires) { //注意：这是公平的tryAcquire()
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //需要先判断自己是不是队头的节点
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

final boolean tryAcquire(int acquires) { //注意：这是非公平的tryAcquire()
    //获取当前线程
    final Thread current = Thread.currentThread();
    //获取state变量值
    int c = getState();
    if (c == 0) { //没有线程占用锁
        //没有 !hasQueuedPredecessors() 判断，不考虑自己是不是在队头，直接申请锁
        if (compareAndSetState(0, acquires)) {
            //占用锁成功,设置独占线程为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) { //当前线程已经占用该锁
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 更新state值为新的重入次数
        setState(nextc);
        return true;
    }
    //获取锁失败
    return false;
```

4.3 addWaiter()

当前线程加入AQS双向链表队列。

写入之前需要将当前线程包装为一个 Node 对象(addWaiter(Node.EXCLUSIVE))。

首先判断队列是否为空，不为空时则将封装好的 Node 利用 CAS 写入队尾，如果出现并发写入失败就需要调用 enq(node); 来写入了。

```java
/**
 * 将新节点和当前线程关联并且入队列
 * @param mode 独占/共享
 * @return 新节点
 */
private Node addWaiter(Node mode) {
    //初始化节点,设置关联线程和模式(独占 or 共享)
    Node node = new Node(Thread.currentThread(), mode);
    // 获取尾节点引用
    Node pred = tail;
    // 尾节点不为空,说明队列已经初始化过
    if (pred != null) {
        node.prev = pred;
        // 设置新节点为尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 尾节点为空,说明队列还未初始化,需要初始化head节点并入队新节点
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

enq()的处理逻辑就相当于`自旋`加上`CAS`保证一定能写入队列。

4.4 acquireQueued()

写入队列后，需要挂起当前线程

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

首先会根据 **node.predecessor()** 获取到上一个节点是否为头节点，如果是则尝试获取一次锁，获取成功就万事大吉了。

如果不是头节点，或者获取锁失败，则会根据上一个节点的 waitStatus 状态来处理(shouldParkAfterFailedAcquire(p, node))。

waitStatus 用于记录当前节点的状态，如节点取消、节点等待等。

shouldParkAfterFailedAcquire(p, node) 返回当前线程是否需要挂起，如果需要则调用 parkAndCheckInterrupt()：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

他是利用 `LockSupport` 的 `part` 方法来挂起当前线程的，直到被唤醒。



4.5 unlock()

释放时候，state--，通过state==0判断锁是否完全被释放。

成功释放锁的话，唤起一个被挂起的线程

```java
public void unlock() {
    sync.release(1);
}
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
        	   //唤醒被挂起的线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}
//尝试释放锁
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

## 总结

0.每一个ReentrantLock自身维护一个AQS队列记录申请锁的线程信息；

1.通过大量CAS保证多个线程竞争锁的时候的并发安全；

2.可重入的功能是通过维护state变量来记录重入次数实现的。

3.公平锁需要维护队列，通过AQS队列的先后顺序获取锁，缺点是会造成大量线程上下文切换；

4.非公平锁可以直接抢占，所以效率更高；