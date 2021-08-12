

## synchronized

把面试中遇到的问题进行了整理. 本篇文章copy+整理自:

​    \1. http://www.cnblogs.com/lingepeiyong/archive/2012/10/30/2745973.html

​    \2. http://www.cnblogs.com/paddix/p/5405678.html

​    \3. https://blog.csdn.net/javazejian/article/details/72828483



### 请描述synchronized底层语义以及原理

Java 虚拟机中的同步(Synchronization)基于进入和退出管程(Monitor)对象实现



### 你是怎么知道monitorenter的?

用javap命令进行反编译. 比如有这样一个java源代码: Main.java

```
javac Main.java  javap -c Main.class
```

 就可以看到反编译的代码了.



### 方法级的synchronized也是根据monitor实现的吗?

方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会 检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放。



### 可重入锁的实现原理？

重入锁实现可重入性原理或机制是：每一个锁关联一个线程持有者和计数器，当计数器为 0 时表示该锁没有被任何线程持有，那么任何线程都可能获得该锁而调用相应的方法；当某一线程请求成功后，JVM会记下锁的持有线程，并且将计数器置为 1；此时其它线程请求该锁，则必须等待；而该持有锁的线程如果再次请求这个锁，就可以再次拿到这个锁，同时计数器会递增；当线程退出同步代码块时，计数器会递减，如果计数器为 0，则释放该锁。



### 描述一下等待唤醒机制与synchronized的联系?

所谓等待唤醒机制本篇主要指的是notify/notifyAll和wait方法，在使用这3个方法时，必须处于synchronized代码块或者synchronized方法中，否则就会抛出IllegalMonitorStateException异常，这是因为调用这几个方法前必须拿到当前对象的监视器monitor对象，也就是说notify/notifyAll和wait方法依赖于monitor对象，在前面的分析中，我们知道monitor 存在于对象头的Mark Word 中(存储monitor引用指针)，而synchronized关键字可以获取 monitor ，这也就是为什么notify/notifyAll和wait方法必须在synchronized代码块或者synchronized方法调用的原因。



### wait()和sleep的区别?

与sleep方法不同的是, wait方法调用完成后，线程将被暂停，但wait方法将会释放当前持有的监视器锁(monitor)，直到有线程调用notify/notifyAll方法后方能继续执行，而sleep方法只让线程休眠并不释放锁。同时notify/notifyAll方法调用后，并不会马上释放监视器锁，而是在相应的synchronized(){}/synchronized方法执行结束后才自动释放锁。



### 什么是虚假唤醒？

举个例子，我们现在有一个生产者-消费者队列和三个线程。

1） 1号线程从队列中获取了一个元素，此时队列变为空。

2） 2号线程也想从队列中获取一个元素，但此时队列为空，2号线程便只能进入阻塞(cond.wait())，等待队列非空。

3） 这时，3号线程将一个元素入队，并调用cond.notify()唤醒条件变量。

4） 处于等待状态的2号线程接收到3号线程的唤醒信号，便准备解除阻塞状态，执行接下来的任务(获取队列中的元素)。

5） 然而可能出现这样的情况：当2号线程准备获得队列的锁，去获取队列中的元素时，此时1号线程刚好执行完之前的元素操作，返回再去请求队列中的元素，1号线程便获得队列的锁，检查到队列非空，就获取到了3号线程刚刚入队的元素，然后释放队列锁。

6） 等到2号线程获得队列锁，判断发现队列仍为空，1号线程“偷走了”这个元素，所以对于2号线程而言，这次唤醒就是“虚假”的，它需要再次等待队列非空。



### 描述一下Monitor中的内部队列的作用?

ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。如下图所示

![img](https://images2018.cnblogs.com/blog/1251417/201806/1251417-20180616170704850-1675762352.png)



### 哪些对象可以作为synchronized锁?

Java中任意对象可以作为锁. monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因(关于这点稍后还会进行分析)

一般而言，synchronized使用的锁对象是存储在Java对象头里的，jvm中采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来的1个字记录的是数组长度)



### 对象头的结构是什么样的?

Hotspot中对象在内存中的结构：

![img](https://img2018.cnblogs.com/blog/1251417/201908/1251417-20190827112724292-1470020602.png)

从上面的这张图里面可以看出，对象在内存中的结构主要包含以下几个部分：

   \1. Mark Word：对象的Mark Word部分占4个字节，其内容是一系列的标记位，比如轻量级锁的标记位，偏向锁标记位等等。

   \2. Class对象指针：Class对象指针的大小也是4个字节，其指向的位置是对象对应的Class对象（其对应的元数据对象）的内存地址

   \3. 对象实际数据：这里面包括了对象的所有成员变量，其大小由各个成员变量的大小决定，比如：byte和boolean是1个字节，short和char是2个字节，int和float是4个字节，long和double是8个字节，reference是4个字节

   \4. 对齐：最后一部分是对齐填充的字节，按8个字节填充。

根据上面的图，那么我们可以得出Integer的对象的结构如下：

![img](https://img2018.cnblogs.com/blog/1251417/201908/1251417-20190827112845171-251088390.png)

 

Integer只有一个int类型的成员变量value，所以其对象实际数据部分的大小是4个字节，然后再在后面填充4个字节达到8字节的对齐，所以可以得出Integer对象的大小是16个字节。

因此，我们可以得出**Integer对象的大小是原生的int类型的4倍**。

关于对象的内存结构，需要注意数组的内存结构和普通对象的内存结构稍微不同，因为数据有一个长度length字段，所以在对象头后面还多了一个int类型的length字段，占4个字节，接下来才是数组中的数据，如下图：

![img](https://img2018.cnblogs.com/blog/1251417/201908/1251417-20190827112906739-940485026.png)

 

对象头中的MarkWord 结构如下: 

| **锁状态** | 25 bit                       | 4bit         | 1bit         | 2bit |      |
| ---------- | ---------------------------- | ------------ | ------------ | ---- | ---- |
| 23bit      | 2bit                         | 是否是偏向锁 | 锁标志位     |      |      |
| 轻量级锁   | 指向栈中锁记录的指针         | 00           |              |      |      |
| 重量级锁   | 指向互斥量（重量级锁）的指针 | 10           |              |      |      |
| GC标记     | 空                           | 11           |              |      |      |
| 偏向锁     | 线程ID                       | Epoch        | 对象分代年龄 | 1    | 01   |
| 无锁       | 对象的hashCode               | 对象分代年龄 | 0            | 01   |      |



### synchronized实现可重入的count存在哪儿?

ObjectMonitor在openjdk中的源码路径: openjdk/hotspot/src/share/vm/runtime/objectMonitor.hpp 

数据结构里有_count一项, 用于在重入时进行自增操作, 释放时进行自减操作.

```java
// initialize the monitor, exception the semaphore, all other fields// are simple integers or pointersObjectMonitor() { _header    = NULL; _count    = 0; _waiters   = 0, _recursions  = 0; _object    = NULL; _owner    = NULL; _WaitSet   = NULL; _WaitSetLock = 0 ; _Responsible = NULL ; _succ     = NULL ; _cxq     = NULL ; FreeNext   = NULL ; _EntryList  = NULL ; _SpinFreq   = 0 ; _SpinClock  = 0 ; OwnerIsThread = 0 ; _previous_owner_tid = 0;}
```

## Java虚拟机对synchronized的优化

锁的状态总共有四种，无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁，但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级.



### 自旋锁与自适应自旋

通常我们称Sychronized锁是一种重量级锁，是因为在互斥状态下，没有得到锁的线程会被挂起阻塞，而挂起线程和恢复线程的操作都需要转入内核态中完成。同时，虚拟机开发团队也注意到，许多应用上的数据锁只会持续很短的一段时间，如果为了这段时间去挂起和恢复线程是不值得的，所以引入了自旋锁。所谓的自旋，就是让没有获得锁的线程自己运行一段时间的自循环，这就是自旋锁。自旋锁可以通过-XX:+UseSpinning参数来开启。

但这显然并不是最好的一种方法，不挂起线程的代价就是该线程会一直占用处理器。如果锁被占用的时间很短，自旋等待的效果就会很好，反之，自旋会消耗大量处理器资源。因此，自旋的等待时间必须有一定的限度，如果超过限度还没有获得锁，就要挂起线程，这个限度默认是10次，可以使用-XX：PreBlockSpin改变。

在JDK6以后又引入了自适应自旋锁，也就说自旋的时间限度不是一个固定值了，而是由上一次同一个锁的自旋时间及锁的拥有者状态来决定。虚拟机认为，如果同一个锁对象自旋刚刚成功获得锁，那么下一次很可能获得锁，所以允许这次自旋锁自旋很长时间、而如果某个锁很少获得锁，那么以后在获取锁的过程中可能忽略到自旋过程。



### 偏向锁

偏向锁是Java 6之后加入的新锁，它是一种针对加锁操作的优化手段，经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了减少同一线程获取锁(会涉及到一些CAS操作,耗时)的代价而引入偏向锁。偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提供程序的性能。所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。下面我们接着了解轻量级锁。

偏向锁实际上是一种锁优化的，其目的是为了减少数据在无竞争情况下的性能消耗。其核心思想就是锁会偏向第一个获取它的线程，在接下来的执行过程中该锁没有其他的线程获取，则持有偏向锁的线程永远不需要再进行同步。

####     偏向锁的获取

当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里储存锁偏向的线程ID。以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需要检查当前Mark Word中储存的线程是否指向当前线程，如果成功，表示已经获得对象锁；如果检测失败，则需要再测试一下Mark Word中偏向锁的标志是否已经被置为1（表示当前锁是偏向锁）：如果没有则使用CAS操作竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

**偏向锁获取过程：**

　　（1）访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01——确认为可偏向状态。

　　（2）如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤（5），否则进入步骤（3）。

　　（3）如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行（5）；如果竞争失败，执行（4）。

　　（4）如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。

　　（5）执行同步代码。

####     偏向锁的撤销

偏向锁使用一种等待竞争出现才释放锁的机制，所以当有其他线程尝试获得锁时，才会释放锁。偏向锁的撤销，需要等到安全点。它首先会暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果不处于活动状态，则将对象头设置为无锁状态；如果依然活动，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向其他线程，要么恢复到无锁或者标记对象不合适作为偏向锁（膨胀为轻量级锁），最后唤醒暂停的线程。

 

**偏向锁的释放：**

　　偏向锁的撤销在上述第四步骤中有提到**。**偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

####     关闭偏向锁

偏向锁在Java运行环境中默认开启，但是不会随着程序启动立即生效，而是在启动几秒种后才激活，可以使用参数关闭延迟：

-XX：BiasedLockingStartupDelay=0 

同样可以关闭偏向锁

 -XX：UseBiasedLocking=false,那么程序默认进入轻量级锁。

####     **偏向锁升级为轻量级锁**

 ![img](https://images2015.cnblogs.com/blog/820406/201604/820406-20160424163618101-624122079.png)



### 轻量级锁

倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段(1.6之后加入的)，此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。

轻量级锁是JDK1.6之中加入的新型锁机制，它并不是来代替重量级锁的，他的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。

####     轻量级锁加锁

线程在执行同步块之前，JVM会现在当前线程的栈帧中创建用于储存锁记录的空间（LockRecord），并将对象头的Mark Word信息复制到锁记录中。然后线程尝试使用CAS将对象头的MarkWord替换为指向锁记录的指针。如果成功，当前线程获得锁，并且对象的锁标志位转变为“00”，如果失败，表示其他线程竞争锁，当前线程便会尝试自旋获取锁。如果有两条以上的线程竞争同一个锁，那么轻量级锁就不再有效，要膨胀为重量级锁，锁标志的状态变为“10”，MarkWord中储存的就是指向重量级锁（互斥量）的指针，后面等待的线程也要进入阻塞状态。

 

**轻量级锁的加锁过程**

　　（1）在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。这时候线程堆栈与对象头的状态如图2.1所示。

　　（2）拷贝对象头中的Mark Word复制到锁记录中。

　　（3）拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock record里的owner指针指向object mark word。如果更新成功，则执行步骤（3），否则执行步骤（4）。

　　（4）如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态，这时候线程堆栈与对象头的状态如图2.2所示。

　　（5）如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。 而当前线程便尝试使用自旋来获取锁，自旋就是为了不让线程阻塞，而采用循环去获取锁的过程。

 ![img](https://images2015.cnblogs.com/blog/820406/201604/820406-20160424105442866-2111954866.png)

​           图2.1 轻量级锁CAS操作之前堆栈与对象的状态

![img](https://images2015.cnblogs.com/blog/820406/201604/820406-20160424105540163-1019388398.png)　　 

​           图2.2 轻量级锁CAS操作之后堆栈与对象的状态

 

 

####     轻量级锁解锁

轻量级锁解锁时，同样通过CAS操作将对象头换回来。如果成功，则表示没有竞争发生。如果失败，说明有其他线程尝试过获取该锁，锁同样会膨胀为重量级锁。在释放锁的同时，唤醒被挂起的线程。

**轻量级锁的解锁过程：**

　　（1）通过CAS操作尝试把线程中复制的Displaced Mark Word对象替换当前的Mark Word。

　　（2）如果替换成功，整个同步过程就完成了。

　　（3）如果替换失败，说明有其他线程尝试过获取该锁（此时锁已膨胀），那就要在释放锁的同时，唤醒被挂起的线程。



### 锁消除

消除锁是虚拟机另外一种锁的优化，这种优化更彻底，Java虚拟机在JIT编译时(可以简单理解为当某段代码即将第一次被执行时进行编译，又称即时编译)，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过这种方式消除没有必要的锁，可以节省毫无意义的请求锁时间，如下StringBuffer的append是一个同步方法，但是在add方法中的StringBuffer属于一个局部变量，并且不会被其他线程所使用，因此StringBuffer不可能存在共享资源竞争的情景，JVM会自动将其锁消除。

```java
public class  StringBufferRemoveSync { 
    public void add(String str1, String str2) {    
        //StringBuffer是线程安全,由于sb只会在append方法中使用,不可能被其他线程引用
        //因此sb属于不可能共享的资源,JVM会自动消除内部的锁    
        StringBuffer sb = new StringBuffer();    
        sb.append(str1).append(str2);  
    }   
    public static void main(String[] args) {    
        StringBufferRemoveSync rmsync = new StringBufferRemoveSync();    
        for (int i = 0; i < 10000000; i++) {      
            rmsync.add("abc", "123");    
        }  
    } 
}
```



###  **锁粗化**

**锁粗化的概念应该比较好理解，就是将多次连接在一起的加锁、解锁操作合并为一次，将多个连续的锁扩展成一个范围更大的锁。举个例子：**

```java
public class StringBufferTest {  
    StringBuffer stringBuffer = new StringBuffer();   
    public void append(){    
        stringBuffer.append("a"); 
        stringBuffer.append("b");    
        stringBuffer.append("c");  
    }
}
```

### synchronized与中断

#### 中断

正如中断二字所表达的意义，**在线程运行(run方法)中间打断它**，在Java中，提供了以下3个有关线程中断的方法：

```java
//中断线程（实例方法）
public void Thread.interrupt();

//判断线程是否被中断（实例方法）
public boolean Thread.isInterrupted();

//判断是否被中断并清除当前中断状态（静态方法）
public static boolean Thread.interrupted();
```

**中断的两种情况：**
一种是当线程处于阻塞状态或者试图执行一个阻塞操作时，我们可以使用实例方法interrupt()进行线程中断，执行中断操作后将会抛出interruptException异常(该异常必须捕捉无法向外抛出)并将**中断状态复位**；

```java
public class InterruputSleepThread3 {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread() {
            @Override
            public void run() {
                //while在try中，通过异常中断就可以退出run循环
                try {
                    while (true) {
                        //当前线程处于阻塞状态，异常必须捕捉处理，无法往外抛出
                        TimeUnit.SECONDS.sleep(2);
                    }
                } catch (InterruptedException e) {
                    System.out.println("Interruted When Sleep");
                    boolean interrupt = this.isInterrupted();
                    //中断状态被复位
                    System.out.println("interrupt:"+interrupt);
                }
            }
        };
        t1.start();
        TimeUnit.SECONDS.sleep(2);
        //中断处于阻塞状态的线程
        t1.interrupt();

        /**
         * 输出结果:
           Interruted When Sleep
           interrupt:false
         */
    }
}
```

> 上述代码中为啥不用Thread.sleep(2000);而用TimeUnit.SECONDS.sleep(2);
> 前者使用时并没有明确的单位说明，而后者非常明确表达秒的单位，事实上后者的内部实现最终还是调用了Thread.sleep(2000);，但为了编写的代码语义更清晰，建议使用TimeUnit.SECONDS.sleep(2);的方式，注意TimeUnit是个枚举类型。

另外一种是当线程处于运行状态时，我们也可调用实例方法interrupt()进行线程中断，但同时必须**手动判断中断状态**，并编写中断线程的代码(其实就是结束run方法体的代码)。有时我们在编码时可能需要兼顾以上两种情况，那么就可以如下编写：

```java
public class InterruputThread {
    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(){
            @Override
            public void run(){
                while(true){
                    //判断当前线程是否被中断，并跳出运行循环
                    if (this.isInterrupted()){
                        System.out.println("线程中断");
                        break;
                    }
                }
			
                System.out.println("已跳出循环,线程中断!");
            }
        };
        t1.start();
        TimeUnit.SECONDS.sleep(2);
        t1.interrupt();//注意非阻塞状态调用interrupt()并不会导致中断状态重置。

        /**
         * 输出结果:
            线程中断
            已跳出循环,线程中断!
         */
    }
}
```

上述两种情况可以兼顾:

```java
public void run(){
    try {
    //判断当前线程是否已中断,注意interrupted方法是静态的,执行后会对中断状态进行复位
    while (!Thread.interrupted()) {
        TimeUnit.SECONDS.sleep(2);
    }
    } catch (InterruptedException e) {

    }
}
```

#### synchronized和中断

上面关于中断说了这么多，其实对于synchronized来说，如果一个线程在等待锁，那么结果只有两种，要么它获得这把锁继续执行，要么它就保存等待，**即使调用中断线程的方法，也不会生效。**
演示代码：

```java
public class SynchronizedBlocked implements Runnable{

    public synchronized void f() {
        System.out.println("Trying to call f()");
        while(true)    // 从未释放锁
            Thread.yield();
    }

    /**
     * 在构造器中创建新线程并启动获取对象锁
     */
    public SynchronizedBlocked() {
        //该线程已持有当前实例锁
        new Thread() {
            public void run() {
                f(); //  Lock acquired by this thread
            }
        }.start();
    }
    public void run() {
        //中断判断
        while (true) {
            if (Thread.interrupted()) {
                System.out.println("中断线程!!");
                break;
            } else {
                f();
            }
        }
    }


    public static void main(String[] args) throws InterruptedException {
        SynchronizedBlocked sync = new SynchronizedBlocked();
        Thread t = new Thread(sync);
        //启动后调用f()方法,无法获取当前实例锁处于等待状态
        t.start();
        TimeUnit.SECONDS.sleep(1);
        //中断线程,无法生效
        t.interrupt();
    }
}
```

我们在SynchronizedBlocked构造函数中创建一个新线程并启动获取调用f()获取到当前实例锁，由于SynchronizedBlocked自身也是线程，启动后在其run方法中也调用了f()，但由于对象锁被其他线程占用，导致t线程只能等到锁，此时我们调用了t.interrupt();但并不能中断线程。

### 等待唤醒

这里我们讲的等待唤醒指的是**notify/notifyAll和wait**方法。在使用这3个方法时，必须处于synchronized代码块或者synchronized方法中，否则就会抛出IllegalMonitorStateException异常，这是因为调用这几个方法前必须拿到当前对象的监视器monitor对象，也就是说notify/notifyAll和wait方法依赖于monitor对象。
我们知道**monitor 存在于对象头的Mark Word** 中(存储monitor引用指针)，而**synchronized关键字可以获取 monitor** ，这也就是为什么notify/notifyAll和wait方法必须在synchronized代码块或者synchronized方法调用的原因。

```java
synchronized (obj) {
       obj.wait();
       obj.notify();
       obj.notifyAll();         
 }
```

与**sleep方法**不同的是wait方法调用完成后，线程将被暂停，但wait方法将会释放当前持有的监视器锁(monitor)，直到有线程调用notify/notifyAll方法后方能继续执行，而sleep方法**只让线程休眠并不释放锁**。

同时notify/notifyAll方法调用后，并不会马上释放监视器锁，而是**在相应的synchronized(){}/synchronized方法执行结束后**才自动释放锁。