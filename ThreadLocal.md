- “知道ThreadLocal吗？”
- “讲讲你对ThreadLocal的理解”

当然了，也有面试官会慢慢引导到这个话题上，比如提问“在多线程环境下，如何防止自己的变量被其它线程篡改”，将主动权交给你自己，剩下的靠自己发挥。

那么ThreadLocal可以做什么，在了解它的应用场景之前，我们先看看它的实现原理，只有知道了实现原理，才好判断它是否符合自己的业务场景。

### ThreadLocal是什么

首先，它是一个数据结构，有点像HashMap，可以保存"key : value"键值对，但是一个ThreadLocal只能保存一个，并且各个线程的数据互不干扰。

```java
ThreadLocal<String> localName = new ThreadLocal();
localName.set("占小狼");
String name = localName.get();
```

在线程1中初始化了一个ThreadLocal对象localName，并通过set方法，保存了一个值`占小狼`，同时在线程1中通过`localName.get()`可以拿到之前设置的值，但是如果在线程2中，拿到的将是一个null。

这是为什么，如何实现？不过之前也说了，ThreadLocal保证了各个线程的数据互不干扰。

看看`set(T value)`和`get()`方法的源码

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

可以发现，每个线程中都有一个`ThreadLocalMap`数据结构，当执行set方法时，其值是保存在当前线程的`threadLocals`变量中，当执行set方法中，是从当前线程的`threadLocals`变量获取。

所以在线程1中set的值，对线程2来说是摸不到的，而且在线程2中重新set的话，也不会影响到线程1中的值，保证了线程之间不会相互干扰。

那每个线程中的`ThreadLoalMap`究竟是什么？

### ThreadLoalMap

本文分析的是1.7的源码。

从名字上看，可以猜到它也是一个类似HashMap的数据结构，但是在ThreadLocal中，并没实现Map接口。

在ThreadLoalMap中，也是初始化一个大小16的Entry数组，Entry对象用来保存每一个key-value键值对，只不过这里的key永远都是ThreadLocal对象，是不是很神奇，通过ThreadLocal对象的set方法，结果把ThreadLocal对象自己当做key，放进了ThreadLoalMap中。

![img](https:////upload-images.jianshu.io/upload_images/2184951-9611b7b31c9b2e20.png?imageMogr2/auto-orient/strip|imageView2/2/w/1068/format/webp)

这里需要注意的是，ThreadLoalMap的Entry是继承WeakReference，和HashMap很大的区别是，Entry中没有next字段，所以就不存在链表的情况了。

### hash冲突

没有链表结构，那发生hash冲突了怎么办？

先看看ThreadLoalMap中插入一个key-value的实现

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

每个ThreadLocal对象都有一个hash值`threadLocalHashCode`，每初始化一个ThreadLocal对象，hash值就增加一个固定的大小`0x61c88647`。

在插入过程中，根据ThreadLocal对象的hash值，定位到table中的位置i，过程如下：
 1、如果当前位置是空的，那么正好，就初始化一个Entry对象放在位置i上；
 2、不巧，位置i已经有Entry对象了，如果这个Entry对象的key正好是即将设置的key，那么重新设置Entry中的value；
 3、很不巧，位置i的Entry对象，和即将设置的key没关系，那么只能找下一个空位置；

这样的话，在get的时候，也会根据ThreadLocal对象的hash值，定位到table中的位置，然后判断该位置Entry对象中的key是否和get的key一致，如果不一致，就判断下一个位置

可以发现，set和get如果冲突严重的话，效率很低，因为ThreadLoalMap是Thread的一个属性，所以即使在自己的代码中控制了设置的元素个数，但还是不能控制其它代码的行为。

### 内存泄露

ThreadLocal可能导致内存泄漏，为什么？
 先看看Entry的实现：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

通过之前的分析已经知道，当使用ThreadLocal保存一个value时，会在ThreadLocalMap中的数组插入一个Entry对象，按理说key-value都应该以强引用保存在Entry对象中，但在ThreadLocalMap的实现中，key被保存到了WeakReference对象中。

这就导致了一个问题，ThreadLocal在没有外部强引用时，发生GC时会被回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。

#### 如何避免内存泄露

既然已经发现有内存泄露的隐患，自然有应对的策略，在调用ThreadLocal的get()、set()可能会清除ThreadLocalMap中key为null的Entry对象，这样对应的value就没有GC Roots可达了，下次GC的时候就可以被回收，当然如果调用remove方法，肯定会删除对应的Entry对象。

如果使用ThreadLocal的set方法之后，没有显示的调用remove方法，就有可能发生内存泄露，所以养成良好的编程习惯十分重要，使用完ThreadLocal之后，记得调用remove方法。

```java
ThreadLocal<String> localName = new ThreadLocal();
try {
    localName.set("占小狼");
    // 其它业务逻辑
} finally {
    localName.remove();
}
```

