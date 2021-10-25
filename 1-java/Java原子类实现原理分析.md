# Java原子类实现原理分析

# **悲观的解决方案（阻塞同步）**

　　我们知道，num++看似简单的一个操作，实际上是由**1.读取 2.加一 3.写入** 三步组成的，这是个复合类的操作（**所以我们之前提到过的volatile是无法解决num++的原子性问题的**），在并发环境下，如果不做任何同步处理，就会有线程安全问题。最直接的处理方式就是**加锁**。

```
synchronized(this){
    num++;
 }
```

　　使用独占锁机制来解决，是一种**悲观的**并发策略，抱着一副“总有刁民想害朕”的态势，每次操作数据的时候都认为别的线程会参与竞争修改，所以直接加锁。同一刻只能有一个线程持有锁，那其他线程就会阻塞。线程的挂起恢复会带来很大的性能开销，尽管jvm对于非竞争性的锁的获取和释放做了很多优化，但是一旦有多个线程竞争锁，频繁的阻塞唤醒，还是会有很大的性能开销的。所以，使用synchronized或其他重量级锁来处理显然不够合理。

# **乐观的解决方案（非阻塞同步）**

　　乐观的解决方案，顾名思义，就是很大度乐观，每次操作数据的时候，都认为别的线程不会参与竞争修改，也不加锁。如果操作成功了那最好；如果失败了，比如中途确有别的线程进入并修改了数据（依赖于冲突检测），也不会阻塞，可以采取一些补偿机制，一般的策略就是反复重试。很显然，这种思想相比简单粗暴利用锁来保证同步要合理的多。

　　鉴于并发包中的原子类其实现机理都差不太多，本章我们就通过AtomicInteger这个原子类来进行分析。我们先来看看对于num++这样的操作AtomicInteger是如何保证其原子性的。

```
 /**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }
```

 我们来分析下incrementAndGet的逻辑：

　　1.先获取当前的value值

　　2.对value加一

　　3.第三步是关键步骤，调用compareAndSet方法来来进行原子更新操作，这个方法的语义是：

　　　　**先检查当前value是否等于current，如果相等，则意味着value没被其他线程修改过，更新并返回true。如果不相等，compareAndSet则会返回false，然后循环继续尝试更新。**

　　compareAndSet调用了Unsafe类的compareAndSwapInt方法

```
/**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     *
     * @param expect the expected value
     * @param update the new value
     * @return true if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```



　　Unsafe的compareAndSwapInt是个native方法，也就是平台相关的。它是基于CPU的CAS指令来完成的。

```
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

# **CAS(Compare-and-Swap)**　　

　　CAS算法是由硬件直接支持来保证原子性的，有三个操作数：**内存位置V、旧的预期值A和新值B，当且仅当V符合预期值A时，CAS用新值B原子化地更新V的值，否则，它什么都不做。**

　　**CAS的ABA问题**

　　当然CAS也并不完美，它存在"ABA"问题，假若一个变量初次读取是A，在compare阶段依然是A，但其实可能在此过程中，它先被改为B，再被改回A，而CAS是无法意识到这个问题的。CAS只关注了比较前后的值是否改变，而无法清楚在此过程中变量的变更明细，这就是所谓的ABA漏洞。 