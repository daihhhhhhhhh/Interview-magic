# Java解决CAS机制中ABA问题的方案

通过对atomic包的分析我们知道了CAS机制，我们在看一下CAS的公式。

```
CAS(V,A,B)
1：V表示内存中的地址
2：A表示预期值
3：B表示要修改的新值
```

CAS的原理就是预期值A与内存中的值相比较，如果相同则将内存中的值改变成新值B。这样比较有两类：

**第一类：如果操作的是基本变量，则比较的是 值 是否相等。**

**第二类：如果操作的是对象的引用，则比较的是对象在 内存的地址 是否相等。**

总结一句话就是：比较并交换。

其实**CAS是Java乐观锁的一种实现机制，**在Java并发包中，大部分类就是通过CAS机制实现的线程安全，它不会阻塞线程，如果更改失败则可以自旋重试，但是它也存在很多问题：

```
1：ABA问题，也就是说从A变成B，然后就变成A，但是并不能说明其他线程并没改变过它，利用CAS就发现不了这种改变。
2：由于CAS失败后会继续重试，导致一致占用着CPU。
```

用一个图来说明ABA的问题。

![Java解决CAS机制中ABA问题的方案](http://p9.pstatp.com/large/pgc-image/0831782aa7f7413bb25c3e8549195dc6)

 

线程1准备利用CAS修改变量值A，但是在修改之前，其他线程已经将A变成了B，然后又变成A，即A->B->A,线程1执行CAS的时候发现仍然为A，所以CAS会操作成功，但是其实目前这个A已经是其他线程修改的了，但是线程1并不知道，最终内存值变成了B，这就导致了ABA问题。

接下来我们看一个关于ABA的例子：

```
public class AtomicMarkableReferenceTest {
 private final static String A = "A";
 private final static String B = "B";
 private final static AtomicReference<String> ar = new AtomicReference<>(A);
 public static void main(String[] args) {
 new Thread(() -> {
 try {
 Thread.sleep(Math.abs((int) (Math.random() * 100)));
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 if (ar.compareAndSet(A, B)) {
 System.out.println("我是线程1,我成功将A改成了B");
 }
 }).start();
 new Thread(() -> {
 if (ar.compareAndSet(A, B)) {
 System.out.println("我是线程2,我成功将A改成了B");
 }
 }).start();
 new Thread(() -> {
 if (ar.compareAndSet(B,A)) {
 System.out.println("我是线程3,我成功将B改成了A");
 }
 }).start();
 }
}
```

上面例子运行结果如下，线程1并不知道线程2和线程3已经改过了值，线程1发现此时还是A则会更改成功，这就是ABA：

![Java解决CAS机制中ABA问题的方案](http://p3.pstatp.com/large/pgc-image/3d2ff951cf5e4081885196b613092f49)

 

所以每种技术都有它的两面性，在解决了一些问题的同时也出现了一些新的问题，在JDK中也为我们提供了两种解决ABA问题的方案，接下来我们就看一下是怎样解决的。

本篇文章的主要内容：

```
1：AtomicMarkableReference 实例和源码解析
2：AtomicStampedReference 实例和源码解析
```

# 一、AtomicMarkableReference实例和源码解析

上面的例子如果利用这个类去实现，会怎样呢？稍微改变上面的代码如下：

```
public class AtomicMarkableReferenceTest {
 private final static String A = "A";
 private final static String B = "B";
 private final static AtomicMarkableReference<String> ar = new AtomicMarkableReference<>(A, false);
 public static void main(String[] args) {
 new Thread(() -> {
 try {
 Thread.sleep(Math.abs((int) (Math.random() * 100)));
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 if (ar.compareAndSet(A, B, false, true)) {
 System.out.println("我是线程1,我成功将A改成了B");
 }
 }).start();
 new Thread(() -> {
 if (ar.compareAndSet(A, B, false, true)) {
 System.out.println("我是线程2,我成功将A改成了B");
 }
 }).start();
 new Thread(() -> {
 if (ar.compareAndSet(B, A, ar.isMarked(), true)) {
 System.out.println("我是线程3,我成功将B改成了A");
 }
 }).start();
 }
}
```

运行结果如下：

![Java解决CAS机制中ABA问题的方案](http://p1.pstatp.com/large/pgc-image/d882c5260c9145e399aec4a467004ea1)

 

是不是解决了这个ABA的问题，AtomicMarkableReference仅仅用一个boolean标记解决了这个问题，那接下来我们进入源码看看它是怎么一种机制。

1：成员变量

```
private volatile Pair<V> pair;
```

定义了一个被关键字volatile修饰的Pair,那Pair是什么对象呢？

```
private static class Pair<T> {
//封装了我们传递的对象
 final T reference;
//这个就是boolean标记
 final boolean mark;
 private Pair(T reference, boolean mark) {
 this.reference = reference;
 this.mark = mark;
 }
 static <T> Pair<T> of(T reference, boolean mark) {
 return new Pair<T>(reference, mark);
 }
}
```

2：构造函数

```
public AtomicMarkableReference(V initialRef, boolean initialMark) {
 pair = Pair.of(initialRef, initialMark);
}
```

这个构造函数就是调用Pair中的of()方法，把我们需要操作的对象和boolean标记传递进去。

那说明以后的操作都是基于Pair这个类进行操作了。那接下来我们看一下它的CAS方法是怎样定义的。

```
//expectedReference表示我们传递的预期值
//newReference表示将要更改的新值
//expectedMark表示传递的预期boolean类型标记
//newMark表示将要更改的boolean类型标记的新值。
public boolean compareAndSet(V expectedReference, V newReference, boolean expectedMark, boolean newMark) {
 Pair<V> current = pair;
 return
 expectedReference == current.reference && expectedMark == current.mark 
&& ((newReference == current.reference && newMark == current.mark) || casPair(current, Pair.of(newReference,  newMark)));
}
```

上面的return后的代码分解后主要有三大逻辑：

**第一个逻辑&&(第二个逻辑 || 第三个逻辑)**

第一个逻辑：预期对象和预期的boolean类型标记必须和内部的Pair中相等

```
 expectedReference == current.reference && expectedMark == current.mark 
```

如果第一个逻辑是true，才能继续往下判断，否则直接返回false。

第二个逻辑：如果这个逻辑为true，就不在执行第三个逻辑了

```
newReference == current.reference && newMark == current.mark
```

如果新的将要更改的对象和新的将要更改的boolean类型的标记和内部Pair的相等，则就不在执行第三个逻辑了。如果为false，则继续往下执行第三个逻辑

第三个逻辑：CAS逻辑

```
casPair(current, Pair.of(newReference, newMark))
```

如果预期的对象和预期的boolean标记和Pair都相等，但是新的对象和新的boolean标记和Pair不相等，此时需要进行CAS更新了

从上面的讲解大家能不能总结出来它是怎样解决ABA的问题的，现在我们总结以下：

它是通过把操作的对象和一个boolean类型的标记封装成Pair，而Pair有被volatile修饰，说明只要更改其他线程立刻可见，而只有Pair中的两个成员变量都相等，来解决CAS中ABA的问题的。一个伪流程图如下：

![Java解决CAS机制中ABA问题的方案](http://p3.pstatp.com/large/pgc-image/91b68a4fa9514aedbe65ebc8ba290184)

 

# 二、AtomicStampedReference实例和源码解析

上面我们知道了AtomicMarkableReference是通过添加一个boolean类型标记和操作的对象封装成Pair来解决ABA问题的，但是如果想知道被操作对象更改了几次，这个类就无法处理了，因为它仅仅用一个boolean去标记，所以AtomicStampedReference就是解决这个问题的，它通过一个int类型标记来代替boolean类型的标记。

上面的例子更改如下：

```
public class AtomicMarkableReferenceTest {
 private final static String A = "A";
 private final static String B = "B";
 private static AtomicInteger ai = new AtomicInteger(1);
 private final static AtomicStampedReference<String> ar = new AtomicStampedReference<>(A, 1);
 public static void main(String[] args) {
 new Thread(() -> {
 try {
 Thread.sleep(Math.abs((int) (Math.random() * 100)));
 } catch (InterruptedException e) {
 e.printStackTrace();
 }
 if (ar.compareAndSet(A, B, 1,2)) {
 System.out.println("我是线程1,我成功将A改成了B");
 }
 }).start();
 new Thread(() -> {
 if (ar.compareAndSet(A, B, ai.get(),ai.incrementAndGet())) {
 System.out.println("我是线程2,我成功将A改成了B");
 }
 }).start();
 new Thread(() -> {
 if (ar.compareAndSet(B, A, ai.get(),ai.incrementAndGet())) {
 System.out.println("我是线程3,我成功将B改成了A");
 }
 }).start();
 }
}
```

运行结果：

![Java解决CAS机制中ABA问题的方案](http://p3.pstatp.com/large/pgc-image/04f8473658f64f2e93ef98a4776074b5)

 

1：成员变量

```
private volatile Pair<V> pair;
private static class Pair<T> {
 final T reference;
 final int stamp;
 private Pair(T reference, int stamp) {
 this.reference = reference;
 this.stamp = stamp;
 }
 static <T> Pair<T> of(T reference, int stamp) {
 return new Pair<T>(reference, stamp);
 }
}
```

这种结果和AtomicMarkableReference中的Pair结构类似，只不过是把boolean类型标记改成了int类型标记。

2：构造函数

```
public AtomicStampedReference(V initialRef, int initialStamp) {
 pair = Pair.of(initialRef, initialStamp);
}
```

3：CAS方法

```
public boolean compareAndSet(V expectedReference,
 V newReference,
 int expectedStamp,
 int newStamp) {
 Pair<V> current = pair;
 return
 expectedReference == current.reference &&
 expectedStamp == current.stamp &&
 ((newReference == current.reference &&
 newStamp == current.stamp) ||
 casPair(current, Pair.of(newReference, newStamp)));
}
```

![Java解决CAS机制中ABA问题的方案](http://p1.pstatp.com/large/pgc-image/1524de743477410abf92f1dcac07662a)

 

上面分析了JDK中解决CAS中ABA问题的两种解决方案，他们的原理是相同的，就是添加一个标记来记录更改，两者的区别如下：

1：AtomicMarkableReference 利用一个boolean类型的标记来记录，只能记录它改变过，不能记录改变的次数 。

2：AtomicStampedReference 利用一个int类型的标记来记录，它能够记录改变的次数。